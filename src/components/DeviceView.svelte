<script lang="ts">
	import type { ActionInstance } from "$lib/ActionInstance";
	import type { Context } from "$lib/Context";
	import type { DeviceInfo } from "$lib/DeviceInfo";
	import type { Profile } from "$lib/Profile";

	import Key from "./Key.svelte";

	import { inspectedInstance, inspectedParentAction } from "$lib/propertyInspector";

	import { invoke } from "@tauri-apps/api/core";

	export let device: DeviceInfo;
	export let profile: Profile;

	export let selectedDevice: string;

	function handleDragStart({ dataTransfer }: DragEvent, controller: string, position: number) {
		dataTransfer?.setData("controller", controller);
		dataTransfer?.setData("position", position.toString());
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
		return true;
	}

	async function handleDrop({ dataTransfer }: DragEvent, controller: string, position: number) {
		let context = { device: device.id, profile: profile.id, controller, position };
		let array = controller == "Encoder" ? profile.sliders : profile.keys;
		if (dataTransfer?.getData("action")) {
			let action = JSON.parse(dataTransfer?.getData("action"));
			if (array[position]) {
				return;
			}
			array[position] = await invoke("create_instance", { context, action });
			profile = profile;
		} else if (dataTransfer?.getData("controller")) {
			let oldArray = dataTransfer?.getData("controller") == "Encoder" ? profile.sliders : profile.keys;
			let oldPosition = parseInt(dataTransfer?.getData("position"));
			let response: ActionInstance = await invoke("move_instance", {
				source: { device: device.id, profile: profile.id, controller: dataTransfer?.getData("controller"), position: oldPosition },
				destination: context,
				retain: false,
			});
			if (response) {
				array[position] = response;
				oldArray[oldPosition] = null;
				profile = profile;
			}
		}
	}

	async function handlePaste(source: Context, destination: Context) {
		let response: ActionInstance = await invoke("move_instance", { source, destination, retain: true });
		if (response) {
			(destination.controller == "Encoder" ? profile.sliders : profile.keys)[destination.position] = response;
			profile = profile;
		}
	}

	// Accessibility helpers
	let announcementEl: HTMLElement;

	function announceToScreenReader(message: string) {
		if (announcementEl) {
			announcementEl.textContent = message;
		}
	}
</script>

{#key device}
	<!-- Screen reader announcements -->
	<div bind:this={announcementEl} class="sr-only" aria-live="polite" aria-atomic="true"></div>
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<div
		class="flex flex-col"
		class:hidden={$inspectedParentAction || selectedDevice != device.id}
		role="grid"
		tabindex="0"
		aria-label={`${device.name} Stream Deck with ${device.rows * device.columns} keys${
			device.encoders > 0 ? ` and ${device.encoders} encoders` : ""
		}. Use arrow keys to navigate, Enter to select or assign actions.`}
		on:click={() => inspectedInstance.set(null)}
	>
		<div class="flex flex-col" role="rowgroup">
			{#each { length: device.rows } as _, r}
				<div class="flex flex-row" role="row">
					{#each { length: device.columns } as _, c}
						<Key
							context={{ device: device.id, profile: profile.id, controller: "Keypad", position: (r * device.columns) + c }}
							bind:inslot={profile.keys[(r * device.columns) + c]}
							on:dragover={handleDragOver}
							on:drop={(event) => handleDrop(event, "Keypad", (r * device.columns) + c)}
							on:dragstart={(event) => handleDragStart(event, "Keypad", (r * device.columns) + c)}
							{handlePaste}
							size={device.id.startsWith("sd-") && device.rows == 4 && device.columns == 8 ? 192 : 144}
						/>
					{/each}
				</div>
			{/each}
		</div>

		{#if device.encoders > 0}
			<div class="flex flex-col" role="rowgroup">
				<div class="flex flex-row" role="row">
			{#each { length: device.encoders } as _, i}
				<Key
					context={{ device: device.id, profile: profile.id, controller: "Encoder", position: i }}
					bind:inslot={profile.sliders[i]}
					on:dragover={handleDragOver}
					on:drop={(event) => handleDrop(event, "Encoder", i)}
					on:dragstart={(event) => handleDragStart(event, "Encoder", i)}
					{handlePaste}
					size={device.id.startsWith("sd-") && device.rows == 4 && device.columns == 8 ? 192 : 144}
				/>
			{/each}
				</div>
			</div>
		{/if}
	</div>
{/key}

<style>
	.sr-only {
		position: absolute;
		width: 1px;
		height: 1px;
		padding: 0;
		margin: -1px;
		overflow: hidden;
		clip: rect(0, 0, 0, 0);
		border: 0;
	}
</style>
