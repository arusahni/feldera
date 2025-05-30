<script lang="ts">
  import {
    deletePipeline,
    postPipelineAction,
    type PipelineAction,
    type PipelineStatus,
    type PipelineThumb
  } from '$lib/services/pipelineManager'
  import { join } from 'array-join'
  import { match, P } from 'ts-pattern'
  import { intersect2 } from '$lib/functions/common/array'
  import { useUpdatePipelineList } from '$lib/compositions/pipelines/usePipelineList.svelte'
  import DeleteDialog, { deleteDialogProps } from '$lib/components/dialogs/DeleteDialog.svelte'
  import { useGlobalDialog } from '$lib/compositions/useGlobalDialog.svelte'
  import { isPipelineEditable } from '$lib/functions/pipelines/status'
  import { useToast } from '$lib/compositions/useToastNotification'
  let {
    pipelines,
    selectedPipelines = $bindable()
  }: { pipelines: PipelineThumb[]; selectedPipelines: string[] } = $props()
  const { updatePipelines } = useUpdatePipelineList()
  const availableActions = [
    'start' as const,
    'pause' as const,
    'shutdown' as const,
    'delete' as const
  ]
  let statusActions = (status: PipelineStatus) =>
    match(status)
      .returnType<(typeof availableActions)[number][]>()
      .with(
        { Queued: P.any },
        { CompilingSql: P.any },
        { SqlCompiled: P.any },
        { CompilingRust: P.any },
        (cause) => [
          ...(Object.values(cause)[0].cause === 'upgrade' ? ['shutdown' as const] : []),
          'delete'
        ]
      )
      .with('Shutdown', () => ['start', 'delete'])
      .with('Preparing', 'Provisioning', 'Initializing', () => ['shutdown', 'delete'])
      .with('Running', () => ['shutdown', 'pause'])
      .with('Pausing', () => ['shutdown', 'delete'])
      .with('Paused', () => ['shutdown', 'start'])
      .with('Resuming', () => ['shutdown', 'delete'])
      .with('ShuttingDown', () => ['shutdown'])
      .with('Unavailable', () => ['shutdown', 'delete'])
      .with('SqlError', 'RustError', 'SystemError', () => ['delete'])
      .with({ PipelineError: P.any }, () => ['shutdown', 'delete'])
      .exhaustive()
  let selected = $derived(
    join(
      selectedPipelines,
      pipelines,
      (name) => name,
      (pipeline) => pipeline.name,
      (_, p) => p
    )
  )
  let actions = $derived(
    selected
      .map((p) => p.status)
      .map(statusActions)
      .reduce(
        (acc, cur) =>
          intersect2(
            acc,
            cur,
            (e) => e,
            (e) => e,
            (a) => a
          ),
        availableActions
      )
      .map((action) =>
        match(action)
          .with('start', () => btnStart)
          .with('pause', () => btnPause)
          .with('shutdown', () => btnShutdown)
          .with('delete', () => btnDelete)
          .exhaustive()
      )
  )
  const globalDialog = useGlobalDialog()
  let postPipelinesAction = (action: PipelineAction) => {
    selectedPipelines.forEach((pipelineName) => postPipelineAction(pipelineName, action))
    selectedPipelines = []
  }
  const { toastError } = useToast()
  let deletePipelines = () => {
    selected.forEach(async (pipeline) => {
      if (!isPipelineEditable(pipeline.status)) {
        await postPipelineAction(pipeline.name, 'shutdown').then((waitFor) =>
          waitFor().catch(toastError)
        )
      }
      return deletePipeline(pipeline.name)
    })
    updatePipelines((ps) => ps.filter((p) => !selectedPipelines.includes(p.name)))
    selectedPipelines = []
  }
</script>

{#snippet btnStart()}
  <button class="btn preset-tonal-surface" onclick={() => postPipelinesAction('start')}>
    <span class="fd fd-play text-[20px]"></span>
    Start
  </button>
{/snippet}
{#snippet btnPause()}
  <button class="btn preset-tonal-surface" onclick={() => postPipelinesAction('pause')}>
    <span class="fd fd-pause text-[20px]"></span>
    Pause
  </button>
{/snippet}
{#snippet btnShutdown()}
  <button class="btn preset-tonal-surface" onclick={() => (globalDialog.dialog = shutdownDialog)}>
    <span class="fd fd-square text-[20px]"></span>
    Shutdown
  </button>
{/snippet}
{#snippet btnDelete()}
  <button class="btn preset-tonal-surface" onclick={() => (globalDialog.dialog = deleteDialog)}>
    <span class="fd fd-trash-2 text-[20px]"></span>
    Delete
  </button>
{/snippet}

{#each actions as action}
  {@render action()}
{/each}

{#snippet deleteDialog()}
  <DeleteDialog
    {...deleteDialogProps(
      'Delete',
      () =>
        selectedPipelines.length === 1
          ? '1 pipeline'
          : selectedPipelines.length.toFixed() + ' pipelines',
      deletePipelines
    )()}
    onClose={() => (globalDialog.dialog = null)}
  ></DeleteDialog>
{/snippet}

{#snippet shutdownDialog()}
  <DeleteDialog
    {...deleteDialogProps(
      'Shutdown',
      () =>
        selectedPipelines.length === 1
          ? '1 pipeline'
          : selectedPipelines.length.toFixed() + ' pipelines',
      () => {
        return postPipelinesAction('shutdown')
      },
      selectedPipelines.length === 1
        ? 'The internal state of the pipeline will be reset.'
        : 'The internal state of these pipelines will be reset.'
    )()}
    onClose={() => (globalDialog.dialog = null)}
  ></DeleteDialog>
{/snippet}
