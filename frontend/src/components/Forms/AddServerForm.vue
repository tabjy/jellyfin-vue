<template>
  <div>
    <v-form
      v-model="valid"
      :disabled="loading"
      @submit.prevent="connectToServer">
      <v-text-field
        v-model="serverUrl"
        variant="outlined"
        autofocus
        :label="$t('login.serverAddress')"
        type="url"
        :rules="rules" />
      <v-row align="center" no-gutters>
        <v-col v-if="previousServerLength" class="mr-2">
          <v-btn
            block
            size="large"
            variant="elevated"
            @click="router.push('/server/select')">
            {{ $t('login.changeServer') }}
          </v-btn>
        </v-col>
        <v-col class="mr-2">
          <v-btn
            :disabled="!valid"
            :loading="loading"
            block
            size="large"
            color="primary"
            variant="elevated"
            type="submit">
            {{ $t('login.connect') }}
          </v-btn>
        </v-col>
      </v-row>
    </v-form>
  </div>
</template>

<script setup lang="ts">
import { ref, unref } from 'vue';
import { useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import { useRemote } from '@/composables';

const remote = useRemote();
const router = useRouter();
const i18n = useI18n();
const valid = ref(false);
const previousServerLength = unref(remote.auth.servers.length);
const serverUrl = ref('');
const loading = ref(false);

const rules = [
  (v: string): boolean | string => !!v.trim() || i18n.t('validation.required')
];

/**
 * Attempts a connection to the given server
 */
async function connectToServer(): Promise<void> {
  loading.value = true;

  try {
    await remote.auth.connectServer(serverUrl.value);

    if (previousServerLength === 0) {
      await router.push('/server/login');
    } else {
      await router.push('/server/select');
    }
  } finally {
    loading.value = false;
  }
}
</script>
