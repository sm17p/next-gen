<script lang="ts">
	import { linear } from "svelte/easing";
	import { fade, type TransitionConfig } from "svelte/transition";
	import { previewCtx } from "./preview-ctx.svelte";
	import type { Snippet } from "svelte";

	const { values, schema } = previewCtx.get();

	interface Props {
		children: Snippet;
		class?: string;
	}
	const { children, class: className }: Props = $props();

	let open = $state(false);

	function fix<El extends HTMLElement | SVGElement>(
		node: El,
		fn: (node: El) => TransitionConfig,
	): TransitionConfig {
		const config = fn(node);
		if (!config.delay) return config;

		const easing = config.easing ?? linear;
		const delay = config.delay;
		const duration = config.duration ?? 0;
		const newDuration = duration + delay;

		const getTimingValues = (t: number) => {
			const transpired = t * newDuration;
			const withoutDelay = Math.max(0, transpired - delay);
			const actualT = withoutDelay / duration;
			const easedT = easing(actualT);

			return [easedT, 1 - easedT];
		};

		const css = (_t: number) => {
			const [t, u] = getTimingValues(_t);
			return config.css?.(t, u) ?? "";
		};

		const tick = (_t: number) => {
			const [t, u] = getTimingValues(_t);
			return config.tick?.(t, u) ?? Promise.resolve();
		};

		return {
			...config,
			delay: 0,
			duration: newDuration,
			css,
			tick,
			easing: linear,
		};
	}
</script>

<div
	class="not-content relative grid min-h-[500px] place-items-center overflow-clip rounded-2xl border
	bg-gray-100 dark:border-gray-700 dark:bg-gray-950 {className}"
>
	<div class="w-full min-w-0 overflow-clip p-4">
		{@render children()}
	</div>

	{#if !open && values}
		<button
			class="absolute bottom-4 left-4 z-10 cursor-pointer rounded-lg bg-gray-500 px-2 py-1
		text-sm text-white transition hover:bg-gray-700 active:bg-gray-800"
			onclick={() => (open = !open)}
			in:fix={(el) => fade(el, { delay: 300, duration: 200 })}
			out:fade={{ duration: 100 }}
		>
			Edit props
		</button>
	{/if}

	{#if values}
		<div
			class="absolute bottom-2 left-2 top-2 z-50 w-[200px] rounded-xl border border-gray-300 bg-gray-100
		p-3 shadow-xl backdrop-blur-xl dark:border-none dark:bg-gray-800/80"
			data-preview
			data-open={open}
		>
			<div class="flex items-center justify-between">
				<p class="text-xl font-bold text-black dark:text-white">Props</p>
				<button
					class="cursor-pointer rounded-lg bg-gray-500 px-2 py-1 text-sm
				text-white transition hover:bg-gray-600 active:bg-gray-700"
					onclick={() => (open = !open)}
				>
					Close
				</button>
			</div>

			<hr class="mt-2 block h-[2px] rounded-full bg-gray-300/50 dark:bg-gray-600" />

			<div class="mt-2 flex flex-col gap-2">
				{#each Object.keys(values ?? {}) as key}
					{@const control = schema[key]}
					<label class="flex w-full flex-col items-start gap-1 text-sm font-medium">
						{control.label}
						{#if control.type === "boolean"}
							<input type="checkbox" bind:checked={values[key] as boolean} />
						{:else if control.type === "select"}
							<select
								bind:value={values[key] as string}
								class="self-stretch rounded-md px-1 py-0.5 dark:bg-gray-900"
							>
								{#each control.options as option}
									<option value={option}>{option}</option>
								{/each}
							</select>
						{:else if control.type === "number"}
							<input
								type="number"
								bind:value={() => values[key] as number,
							(v: number) => {
								// Make sure that inputed values don't go outside the specified min/max values.
								if (control.min && v < control.min) {
									values[key] = control.min;
								} else if (control.max && v > control.max) {
									values[key] = control.max;
								} else {
									values[key] = v;
								}
							}}
								min={control.min}
								max={control.max}
								class="self-stretch rounded-md px-1 py-0.5 dark:bg-gray-900"
							/>
						{:else if control.type === "string"}
							<input
								type="text"
								bind:value={values[key] as string}
								class="self-stretch rounded-md px-1 py-0.5 dark:bg-gray-900"
							/>
						{/if}
					</label>
				{/each}
			</div>
		</div>
	{/if}
</div>

<style>
	[data-preview][data-open="false"] {
		scale: 0.95;
		translate: -110%;
		opacity: 0.75;
		transition:
			opacity 0.15s var(--ease-out-quad),
			scale 0.15s var(--ease-out-quad),
			translate 0.15s var(--ease-out-quad) 0.15s;
	}

	[data-preview][data-open="true"] {
		transition:
			opacity 0.15s var(--ease-out-quad) 0.15s,
			scale 0.15s var(--ease-out-quad) 0.15s,
			translate 0.15s var(--ease-out-quad);
	}
</style>
