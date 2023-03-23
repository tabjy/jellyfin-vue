<template>
  <v-col v-if="lyrics" cols="12" class="pa-0 d-none d-md-inline">
    <div class="d-flex flex-column justify-center">
      <div class="d-flex align-center justify-center">
        <span class="mb-1">{{ currentLine || '' }}</span>
      </div>
    </div>
  </v-col>
</template>

<script lang="ts">
import Vue from 'vue';
import { mapStores } from 'pinia';
import { BaseItemDto } from '@jellyfin/client-axios';
import { playbackManagerStore } from '~/store';
import { ticksToMs, msToTicks } from '~/utils/time';

const NETEASE_API_GATEWAY = 'https://netease-cloud-music-api-tabjy.vercel.app';
const OFFSET = 0; // ms

export default Vue.extend({
  data() {
    return {
      item: undefined as BaseItemDto | undefined,
      lyrics: undefined as Array<{ Start: number; Text: string }> | undefined,
      progress: 0
    };
  },
  computed: {
    ...mapStores(playbackManagerStore),
    currentLine(): string | undefined {
      const tick = msToTicks((this.playbackManager.currentTime || 0) * 1000);

      return [...this.lyrics!]
        .reverse()
        .find((line) => line.Start + OFFSET <= tick)?.Text;
    }
  },
  watch: {
    'playbackManager.getCurrentItem': {
      handler(): void {
        if (this.item?.Id === this.playbackManager.getCurrentItem?.Id) {
          return;
        }

        this.item = this.playbackManager.getCurrentItem;
        this.lyrics = undefined;
        this.getLyrics();
      },
      immediate: true
    }
  },
  methods: {
    async getLyrics(): Promise<void> {
      const item = this.playbackManager.getCurrentItem;

      if (!item) {
        return;
      }

      const candidates = await fetch(
        `${NETEASE_API_GATEWAY}/cloudsearch?type=1&keywords=${encodeURIComponent(
          [
            item.Name,
            /* ...([item.Album] || []), */
            ...(item.Artists || [])
          ].join(' ')
        )}`
      ).then((resp) => {
        if (resp.status >= 400) {
          throw new Error(resp.statusText);
        }

        return resp.json();
      });

      const song = candidates.result.songs
        .map((candidate: { dt: number }) => ({
          ...candidate,
          delta: Math.abs(
            candidate.dt - (ticksToMs(item.RunTimeTicks) || candidate.dt)
          )
        }))
        .filter(
          (candidate: { id: number; delta: number }) =>
            candidate.delta < 5 * 1000
        )
        .slice(0, 10)
        .sort(
          (lhs: { delta: number }, rhs: { delta: number }) =>
            lhs.delta - rhs.delta
        )[0];

      if (!song) {
        return;
      }

      const lyrics = await fetch(
        `${NETEASE_API_GATEWAY}/lyric?id=${song.id}`
      ).then((resp) => {
        if (resp.status >= 400) {
          throw new Error(resp.statusText);
        }

        return resp.json();
      });

      if (lyrics?.lrc?.lyric && this.item?.Id === item.Id) {
        this.lyrics = this.parseLrc(lyrics.lrc.lyric);
      }
    },
    parseLrc(lrc: string): Array<{ Start: number; Text: string }> {
      const lineRegex = /((?:\[\d+:\d{2}(?:.\d+)*\])+)(.*)/;
      const timestampRegex = /\[(\d+:\d{2}(?:.\d+)*)\]/g;

      return lrc
        .split('\n')
        .filter((line) => lineRegex.test(line))
        .flatMap((line) => {
          const [_, timestampGroup, text] = line.match(lineRegex)!;
          const timestamps = timestampGroup.match(timestampRegex)!;

          return timestamps.map((timestamp) => {
            const [mm, ss] = timestamp
              .replace('[', '')
              .replace(']', '')
              .split(':');
            const ms =
              (Number.parseInt(mm, 10) * 60 + Number.parseFloat(ss)) * 1000;

            return {
              Start: msToTicks(ms),
              Text: text.trim()
            };
          });
        })
        .sort((lhs, rhs) => lhs.Start - rhs.Start);
    }
  }
});
</script>
