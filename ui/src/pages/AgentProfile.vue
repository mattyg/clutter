<template>
  <q-page class="row" :style-fn="pageHeightCorrection">
    <div class="col-8">
      <h6 class="q-mt-none q-mb-md">Profile</h6>
      <q-spinner-pie v-if="loadingProfile" size="10%" color="primary" />

      <q-card v-else v-bind="$attrs" square class="q-mb-md text-body1">
        <q-card-section class="flex justify-between">
          <div class="flex items-center">
            <agent-avatar
              :agentPubKey="agentPubKey"
              size="50"
              :class="['q-mr-lg', { 'cursor-pointer': !isMyProfile }]"
            />
            <div class="q-mr-lg text-primary text-weight-medium">
              {{ displayName }}
            </div>
            <div class="text-primary">@{{ nickname }}</div>
          </div>
          <ButtonFollow v-if="!isMyProfile" :agentPubKey="agentPubKey" />
        </q-card-section>

        <q-card-section class="flex">
          <div class="q-mr-md">
            <div>
              <label class="text-weight-medium">Bio:</label>
            </div>
            <div>
              <label class="text-weight-medium">Location:</label>
            </div>
          </div>
          <div class="col-grow">
            <div>{{ bio }}</div>
            <div>{{ location }}</div>
          </div>
        </q-card-section>
      </q-card>

      <h6 class="q-mb-md">Mews</h6>
      <EmptyMewsFeed v-if="!loadingMews && mews.length === 0" />
      <MewList
        v-else
        :is-loading="loadingMews"
        :mews="mews"
        :on-toggle-lick-mew="onToggleLickMew"
        :on-publish-mew="onPublishMew"
      />
    </div>

    <div class="follow-col col self-start q-pl-xl q-pr-md">
      <h6 class="q-mt-none q-mb-md">Following</h6>
      <FolloweesList :agentPubKey="agentPubKey" />
      <h6 class="q-mb-md">Followed by</h6>
      <FollowersList :agentPubKey="agentPubKey" />
    </div>
  </q-page>
</template>

<script setup lang="ts">
import ButtonFollow from "@/components/ButtonFollow.vue";
import EmptyMewsFeed from "@/components/EmptyMewsFeed.vue";
import FolloweesList from "@/components/FolloweesList.vue";
import FollowersList from "@/components/FollowersList.vue";
import {
  getFeedMewAndContext,
  mewsBy,
  myFollowing,
} from "@/services/clutter-dna";
import { useProfilesStore } from "@/services/profiles-store";
import { FeedMew, MewType, MewTypeName, PROFILE_FIELDS } from "@/types/types";
import { isSameHash } from "@/utils/hash";
import { showError, showMessage } from "@/utils/notification";
import { pageHeightCorrection } from "@/utils/page-layout";
import { ActionHash, decodeHashFromBase64 } from "@holochain/client";
import { computed, onMounted, ref, watch } from "vue";
import { useRoute } from "vue-router";
import MewList from "../components/MewList.vue";

const profilesStore = useProfilesStore();
const route = useRoute();
const agentPubKey = computed(() =>
  decodeHashFromBase64(route.params.agent as string)
);
const loadingMews = ref(false);
const loadingProfile = ref(false);
const nickname = ref("");
const displayName = ref("");
const bio = ref("");
const location = ref("");
const isFollowing = ref(false);
const mews = ref<FeedMew[]>([]);

const isMyProfile = computed(() =>
  isSameHash(agentPubKey.value, profilesStore.value.client.client.myPubKey)
);

const loadMews = async () => {
  try {
    loadingMews.value = true;
    mews.value = await mewsBy(agentPubKey.value);
  } catch (error) {
    showError(error);
  } finally {
    loadingMews.value = false;
  }
};

const loadProfile = async () => {
  try {
    loadingProfile.value = true;
    const [profile, currentMyFollowing] = await Promise.all([
      profilesStore.value.client.getAgentProfile(agentPubKey.value),
      myFollowing(),
    ]);
    if (profile) {
      nickname.value = profile.nickname;
      displayName.value = profile.fields[PROFILE_FIELDS.DISPLAY_NAME];
      bio.value = profile.fields[PROFILE_FIELDS.BIO];
      location.value = profile.fields[PROFILE_FIELDS.LOCATION];
    }
    isFollowing.value = currentMyFollowing.includes(agentPubKey.value);
  } catch (error) {
    showError(error);
  } finally {
    loadingProfile.value = false;
  }
};

const load = () => {
  loadProfile();
  loadMews();
};

onMounted(load);

watch(
  () => route.params.agent,
  (value) => {
    if (value) {
      load();
    }
  }
);

const onToggleLickMew = async (hash: ActionHash) => {
  try {
    const index = mews.value.findIndex((mew) =>
      isSameHash(hash, mew.actionHash)
    );
    if (index !== -1) {
      mews.value[index] = await getFeedMewAndContext(hash);
    }
  } catch (error) {
    showError(error);
  }
};

const onPublishMew = async (mewType: MewType) => {
  loadMews();
  showMessage(
    MewTypeName.Reply in mewType
      ? "Replied to mew"
      : MewTypeName.MewMew in mewType
      ? "Mew mewmewed"
      : "Quoted mew"
  );
};
</script>

<style lang="sass">
.follow-col
  position: sticky
  top: $toolbar-min-height + map-get(map-get($spaces, "xl"), "y")
</style>
