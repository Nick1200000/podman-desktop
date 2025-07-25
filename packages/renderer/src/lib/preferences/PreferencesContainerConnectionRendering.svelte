<script lang="ts">
import { Tab } from '@podman-desktop/ui-svelte';
import { Buffer } from 'buffer';
import { onDestroy, onMount } from 'svelte';
import type { Unsubscriber } from 'svelte/store';
import { router } from 'tinro';

import { handleNavigation } from '/@/navigation';
import type { IConfigurationPropertyRecordedSchema } from '/@api/configuration/models.js';
import { NavigationPage } from '/@api/navigation-page';
import type { ProviderConnectionInfo, ProviderContainerConnectionInfo, ProviderInfo } from '/@api/provider-info';

import Route from '../../Route.svelte';
import { providerInfos } from '../../stores/providers';
import IconImage from '../appearance/IconImage.svelte';
import ConnectionErrorInfoButton from '../ui/ConnectionErrorInfoButton.svelte';
import ConnectionStatus from '../ui/ConnectionStatus.svelte';
import DetailsPage from '../ui/DetailsPage.svelte';
import { getTabUrl, isTabSelected } from '../ui/Util';
import { eventCollect } from './preferences-connection-rendering-task';
import PreferencesConnectionActions from './PreferencesConnectionActions.svelte';
import PreferencesConnectionDetailsLogs from './PreferencesConnectionDetailsLogs.svelte';
import PreferencesConnectionDetailsTerminal from './PreferencesConnectionDetailsTerminal.svelte';
import PreferencesContainerConnectionDetailsSummary from './PreferencesContainerConnectionDetailsSummary.svelte';
import type { IConnectionRestart, IConnectionStatus } from './Util';
import { getProviderConnectionName } from './Util';

export let properties: IConfigurationPropertyRecordedSchema[] = [];
export let providerInternalId: string | undefined = undefined;
export let connection: string | undefined = undefined;
export let name: string | undefined = undefined;

const connectionName = Buffer.from(name ?? '', 'base64').toString();
const socketPath: string = Buffer.from(connection ?? '', 'base64').toString();
let connectionStatus: IConnectionStatus;
let noLog = true;
let connectionInfo: ProviderContainerConnectionInfo | undefined;
let providerInfo: ProviderInfo | undefined;
let loggerHandlerKey: symbol | undefined;
let configurationKeys: IConfigurationPropertyRecordedSchema[];
$: configurationKeys = properties
  .filter(property => property.scope === 'ContainerConnection')
  .sort((a, b) => (a.id ?? '').localeCompare(b.id ?? ''));

let providersUnsubscribe: Unsubscriber;
onMount(async () => {
  noLog = true;
  providersUnsubscribe = providerInfos.subscribe(providerInfosValue => {
    const providers = providerInfosValue;
    providerInfo = providers.find(provider => provider.internalId === providerInternalId);
    connectionInfo = providerInfo?.containerConnections?.find(
      connection => connection.endpoint.socketPath === socketPath && connection.name === connectionName,
    );
    if (!connectionInfo) {
      handleNavigation({
        page: NavigationPage.RESOURCES,
      });
      return;
    }
    if (!providerInfo) {
      return;
    }
    const containerConnectionName = getProviderConnectionName(providerInfo, connectionInfo);
    if (containerConnectionName && (!connectionStatus || connectionStatus.status !== connectionInfo.status)) {
      if (loggerHandlerKey !== undefined) {
        connectionStatus = {
          inProgress: true,
          action: 'restart',
          status: connectionInfo.status,
        };
        startContainerProvider(providerInfo, connectionInfo, loggerHandlerKey).catch((err: unknown) =>
          console.error(`Error starting provider ${connectionInfo?.name}`, err),
        );
        loggerHandlerKey = undefined;
      } else {
        connectionStatus = {
          inProgress: false,
          action: undefined,
          status: connectionInfo.status,
        };
      }
    }
    connectionStatus = connectionStatus;
  });
});

onDestroy(() => {
  if (providersUnsubscribe) {
    providersUnsubscribe();
  }
});

async function startContainerProvider(
  provider: ProviderInfo,
  containerConnectionInfo: ProviderContainerConnectionInfo,
  loggerHandlerKey: symbol,
): Promise<void> {
  await window.startProviderConnectionLifecycle(
    provider.internalId,
    containerConnectionInfo,
    loggerHandlerKey,
    eventCollect,
  );
}
function updateConnectionStatus(
  provider: ProviderInfo,
  containerConnectionInfo: ProviderConnectionInfo,
  action?: string,
  error?: string,
): void {
  if (error) {
    if (connectionStatus) {
      connectionStatus = {
        ...connectionStatus,
        inProgress: false,
        error,
      };
    }
  } else if (action) {
    connectionStatus = {
      inProgress: true,
      action: action,
      status: containerConnectionInfo.status,
    };
  }
  connectionStatus = connectionStatus;
}

function addConnectionToRestartingQueue(container: IConnectionRestart): void {
  loggerHandlerKey = container.loggerHandlerKey;
}

function setNoLogs(): void {
  noLog = false;
}
</script>

{#if connectionInfo}
  <DetailsPage title={connectionInfo.displayName}>
    {#snippet subtitleSnippet()}
      {#if connectionInfo}
        <div class="flex flex-row">
          <ConnectionStatus status={connectionInfo.status} />
          <ConnectionErrorInfoButton status={connectionStatus} />
        </div>
      {/if}
    {/snippet}
    {#snippet actionsSnippet()}
      {#if connectionInfo && providerInfo}
        <div class="flex justify-end">
          <PreferencesConnectionActions
            provider={providerInfo}
            connection={connectionInfo}
            connectionStatus={connectionStatus}
            updateConnectionStatus={updateConnectionStatus}
            addConnectionToRestartingQueue={addConnectionToRestartingQueue} />
        </div>
      {/if}
    {/snippet}
    {#snippet iconSnippet()}
      <IconImage image={providerInfo?.images?.icon} alt={providerInfo?.name} class="max-h-10" />
    {/snippet}
    {#snippet tabsSnippet()}
      {#if connectionInfo}
        <Tab title="Summary" selected={isTabSelected($router.path, 'summary')} url={getTabUrl($router.path, 'summary')} />
        {#if connectionInfo.lifecycleMethods && connectionInfo.lifecycleMethods.length > 0}
          <Tab title="Logs" selected={isTabSelected($router.path, 'logs')} url={getTabUrl($router.path, 'logs')} />
        {/if}
        {#if connectionInfo.shellAccess}
          <Tab
            title="Terminal"
            selected={isTabSelected($router.path, 'terminal')}
            url={getTabUrl($router.path, 'terminal')} />
        {/if}
      {/if}
    {/snippet}
    {#snippet contentSnippet()}
      {#if connectionInfo}
        <div class="h-full">
          <Route path="/summary" breadcrumb="Summary" navigationHint="tab">
            <PreferencesContainerConnectionDetailsSummary
              containerConnectionInfo={connectionInfo}
              providerInternalId={providerInternalId}
              properties={configurationKeys} />
          </Route>
          <Route path="/logs" breadcrumb="Logs" navigationHint="tab">
            <PreferencesConnectionDetailsLogs
              providerInternalId={providerInternalId}
              connectionInfo={connectionInfo}
              setNoLogs={setNoLogs}
              noLog={noLog} />
          </Route>
          {#if providerInfo}
            <Route path="/terminal" breadcrumb="Terminal" navigationHint="tab">
              <PreferencesConnectionDetailsTerminal
                provider={providerInfo}
                connectionInfo={connectionInfo}
                screenReaderMode={true} />
            </Route>
          {/if}
        </div>
      {/if}
    {/snippet}
  </DetailsPage>
{/if}
