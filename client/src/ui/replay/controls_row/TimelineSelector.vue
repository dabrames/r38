<!--

Displays the current draft's event history as a scrollable list.

Clicking on an individual entry will jump to that point in the event stream.
In synchronized mode, shows one entry per pick. In temporal mode, each event
gets its own entry.

-->

<template>
  <div class="_timeline-selector">
    <div class="title">Timeline</div>

    <div class="list-cnt">
      <template v-for="entry in listEntries">
        <div
          v-if="entry.type == 'synchronized-header'"
          :key="`round_${entry.round}`"
          class="sync-header"
        >
          Pack {{ entry.round }}
        </div>

        <div
          v-else-if="entry.type == 'synchronized-pick'"
          :key="`synpick-${entry.eventId}`"
          class="sync-pick"
          :class="{
            selected: entry.eventId == currentEventId,
          }"
          @click="onSyncPickClicked(entry.eventId)"
        >
          Pick {{ entry.pick + 1 }}
        </div>

        <div
          v-else-if="entry.type == 'temporal-pick'"
          :key="`tpick-${entry.eventId}`"
          class="temp-pick"
          :class="{
            selected: entry.eventId == currentEventId,
          }"
          @click="onTempPickClicked(entry.eventId, entry.seatId)"
        >
          <div class="event-id">{{ entry.eventId }}</div>
          <div class="temp-pick-message">
            <span class="temp-pick-pack"> p{{ entry.round }}p{{ entry.pick + 1 }} </span>
            {{ entry.playerName }}
            picked
            <span class="picked-card-name">{{ entry.cardName }}</span>
          </div>
        </div>

        <div
          v-else-if="entry.type == 'temporal-other'"
          :key="`tother-${entry.eventId}`"
          class="temp-other"
        >
          <span class="event-id">{{ entry.eventId }}</span>
          Nothing picked
        </div>
      </template>
    </div>

    <div class="synchronized-cnt">
      <input type="checkbox" id="synchronize-checkbox" v-model="synchronizeTimeline" />
      <label for="synchronize-checkbox" class="synchronize-label"> Synchronize timeline </label>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";

import { replayStore, type TimeMode } from "@/state/ReplayStore";
import { draftStore } from "@/state/DraftStore";
import { pushDraftUrlFromState, pushDraftUrlRelative } from "@/router/url_manipulation";
import { isPickEvent } from "@/state/util/isPickEvent";
import { eventToString } from "@/state/util/eventToString";
import { indexOf } from "@/util/collection";
import type { TimelineEvent } from "@/draft/TimelineEvent";

export default defineComponent({
  computed: {
    synchronizeTimeline: {
      get() {
        return replayStore.timeMode == "synchronized";
      },

      set(value: TimeMode) {
        replayStore.setTimeMode(value ? "synchronized" : "original");
        pushDraftUrlFromState(this, draftStore, replayStore);
      },
    },

    listEntries(): ListEntry[] {
      if (replayStore.timeMode == "synchronized") {
        return this.computeSynchronizedList();
      } else {
        return this.computeTemporalList();
      }
    },

    currentEventId(): number {
      const event = replayStore.events[replayStore.eventPos];
      return event != undefined ? event.id : -1;
    },
  },

  methods: {
    onSyncPickClicked(eventId: number) {
      const index = indexOf(replayStore.events, { id: eventId });
      if (index != -1) {
        pushDraftUrlRelative(this, {
          eventIndex: index,
        });
      } else {
        console.warn(`Can't find event ID`, eventId);
      }
    },

    onTempPickClicked(eventId: number, seatId: number) {
      const index = indexOf(replayStore.events, { id: eventId });
      if (index != -1) {
        pushDraftUrlRelative(this, {
          eventIndex: index,
          selection: {
            type: "seat",
            id: seatId,
          },
        });
      } else {
        console.warn(`Can't find event ID`, eventId);
      }
    },

    computeSynchronizedList(): ListEntry[] {
      let currentRound = 0;
      let currentPick = -1;
      const entries = [] as ListEntry[];

      for (const event of replayStore.events) {
        if (event.round != currentRound) {
          entries.push({
            type: "synchronized-header",
            round: event.round,
          });
          currentRound++;
          currentPick = -1;
        }

        if (event.pick != currentPick && isPickEvent(event)) {
          entries.push({
            type: "synchronized-pick",
            pick: event.pick,
            eventId: event.id,
          });
          currentPick = event.pick;
        }
      }

      return entries;
    },

    computeTemporalList() {
      return replayStore.events.map((event) => eventToTemporalListEntry(event));
    },
  },
});

function eventToTemporalListEntry(event: TimelineEvent): ListEntry {
  if (event.type == "pick") {
    // TODO: Support multi-pick and returned cards
    const pickAction = getFirstPickAction(event);

    return {
      type: "temporal-pick",
      eventId: event.id,
      round: event.round,
      seatId: event.associatedSeat,
      pick: event.pick,
      playerName: replayStore.draft.seats[event.associatedSeat].player.name,
      cardName: pickAction.cardName,
    };
  } else if (event.type == "hidden-pick") {
    return {
      type: "temporal-pick",
      eventId: event.id,
      round: event.round,
      seatId: event.associatedSeat,
      pick: event.pick,
      playerName: replayStore.draft.seats[event.associatedSeat].player.name,
      cardName: "a card",
    };
  } else {
    return {
      type: "temporal-other",
      eventId: event.id,
    };
  }
}

function getFirstPickAction(event: TimelineEvent) {
  for (const action of event.actions) {
    if (action.type == "move-card" && action.subtype == "pick-card") {
      return action;
    }
  }
  throw new Error(`Event has ${eventToString(event)} no pick actions`);
}

type ListEntry =
  | SynchronizedHeaderEntry
  | SynchronizedPickEntry
  | TemporalPickEntry
  | TemporalOtherEntry;

interface SynchronizedHeaderEntry {
  type: "synchronized-header";
  round: number;
}

interface SynchronizedPickEntry {
  type: "synchronized-pick";
  pick: number;
  eventId: number;
}

interface TemporalPickEntry {
  type: "temporal-pick";
  eventId: number;
  round: number;
  pick: number;
  playerName: string;
  cardName: string;
  seatId: number;
}

interface TemporalOtherEntry {
  type: "temporal-other";
  eventId: number;
}
</script>

<style scoped>
.title {
  font-size: 14px;
  margin-top: 8px;
  margin-bottom: 8px;
  text-align: center;
}

._timeline-selector {
  display: flex;
  flex-direction: column;
}

.list-cnt {
  flex: 1 0 0;
  overflow-y: scroll;
  padding-top: 10px;
  padding-bottom: 15px;
  user-select: none;
  cursor: default;
  border-top: 1px solid #eaeaea;
  border-bottom: 1px solid #eaeaea;
}

.selected {
  background-color: #fff6cb;
}

.sync-header {
  font-size: 14px;
  color: #a2310d;
  margin-bottom: 4px;
  padding-left: 10px;
}

.sync-header:not(:first-child) {
  margin-top: 10px;
}

.sync-pick {
  font-size: 14px;
  padding: 3px 0;
  padding-left: 10px;
}

.synchronized-cnt {
  padding: 10px 10px;
}

.temp-pick,
.temp-other {
  font-size: 14px;
  padding: 3px 0 3px 5px;
}

.temp-pick {
  display: flex;
}

.event-id {
  display: inline-block;
  min-width: 25px;
  flex: 0 0 auto;
}

.event-id,
.temp-pick-pack,
.temp-other {
  color: #aaa;
}

.picked-card-name {
  color: #1560d4;
}

.temp-pick-message {
  margin-left: 5px;
}
</style>
