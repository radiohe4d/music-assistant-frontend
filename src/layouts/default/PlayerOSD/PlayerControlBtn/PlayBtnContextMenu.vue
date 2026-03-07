<template></template>

<script setup lang="ts">
import api from "@/plugins/api";
import { Player, PlayerFeature, PlayerQueue } from "@/plugins/api/interfaces";
import { ContextMenuItem } from "@layout/default/ItemContextMenu.vue";
import { store } from "@/plugins/store";
import { eventbus } from "@/plugins/eventbus";
import { computed } from "vue";

export interface Props {
  player: Player | undefined;
}

const props = defineProps<Props>();
const contextMenuYOffset: number = -50;

const currentPlayerQueue = computed(() => {
  if (
    props.player &&
    props.player.active_source &&
    props.player.active_source in api.queues
  ) {
    return api.queues[props.player.active_source];
  }
  if (
    props.player &&
    !props.player.active_source &&
    props.player.player_id in api.queues
  ) {
    return api.queues[props.player.player_id];
  }
  return undefined;
});

const openContextMenu = (event: Event) => {
  if (!props.player) return;

  let items = loadMenuItems(props.player, currentPlayerQueue.value);

  showContextMenu(event, items);
};

defineExpose({
  openContextMenu,
});

const loadMenuItems = (
  player: Player,
  playerQueue?: PlayerQueue,
): ContextMenuItem[] => {
  // Default item when no items in the queue.
  let items: ContextMenuItem[] = [
    // Empty Queue
    {
      label: "queue_empty",
      labelArgs: [],
      action: () => null,
      icon: "mdi-not-found",
    },
  ];

  if (!currentPlayerQueue.value?.items) return items;

  items = [
    // Clear Queue
    {
      label: "queue_clear",
      labelArgs: [],
      action: () => {
        api.queueCommandClear(playerQueue!.queue_id);
      },
      icon: "mdi-cancel",
    },

    // Sync Player With
    {
      label: "sync_player_with",
      labelArgs: [],
      icon: "mdi-link-variant",
      subItems: getSyncablePlayers(player)
        .map((p) => {
          return {
            label: p.name,
            labelArgs: [],
            action: () => {
              api.playerCommandGroup(player.player_id, p.player_id);
            },
          };
        })
        .sort((a, b) =>
          a.label.toUpperCase() > b.label?.toUpperCase() ? 1 : -1,
        ),
    },

    // Transfer Queue
    {
      label: "transfer_queue",
      icon: "mdi-swap-horizontal",
      subItems: Object.values(api.players)
        .filter(
          (p) =>
            p.player_id != playerQueue!.queue_id &&
            p.player_id != player.player_id &&
            p.available &&
            p.enabled &&
            !p.synced_to,
        )
        .map((p) => {
          return {
            label: p.name,
            labelArgs: [],
            action: () => {
              api.queueCommandTransfer(playerQueue!.queue_id, p.player_id);
              store.activePlayerId = p.player_id;
            },
          };
        })
        .sort((a, b) =>
          a.label.toUpperCase() > b.label?.toUpperCase() ? 1 : -1,
        ),
    },
  ];

  return items;
};

const getSyncablePlayers = (player: Player) => {
  return Object.values(api.players).filter(
    (p) =>
      p.player_id != player.player_id &&
      p.available &&
      p.enabled &&
      (p.can_group_with.includes(player.provider) ||
        p.can_group_with.includes(player.player_id)) &&
      p.supported_features.includes(PlayerFeature.SET_MEMBERS) &&
      !p.synced_to,
  );
};

const showContextMenu = (event: Event, items: ContextMenuItem[]) => {
  eventbus.emit("contextmenu", {
    items: items,
    posX: (event as PointerEvent).clientX,
    posY: (event as PointerEvent).clientY + contextMenuYOffset,
  });
};
</script>
