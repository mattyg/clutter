<template>
  <q-item :inset-level="contentInsetLevel" class="items-start">
    <q-item-section avatar>
      <AvatarWithPopup :agentPubKey="feedMew.action.author" />
    </q-item-section>

    <q-item-section>
      <div class="row q-mb-sm">
        <div
          :class="{
            'cursor-pointer': !isCurrentProfile(feedMew.action.author),
          }"
          @click="onAgentClick(feedMew.action.author)"
        >
          <span class="q-mr-xs text-primary text-weight-bold">
            {{ agentProfile?.fields[PROFILE_FIELDS.DISPLAY_NAME] }}
          </span>
          <span>@{{ agentProfile?.nickname }}</span>
        </div>

        <span v-if="!isOriginal" class="q-ml-md">
          <q-skeleton
            v-if="loadingOriginalMewAuthor"
            type="text"
            width="4rem"
          />
          <router-link
            v-else-if="originalMew && originalMewAuthor"
            :to="{
              name: ROUTES.profiles,
              params: { agent: encodeHashToBase64(originalMew.action.author) },
            }"
            class="text-secondary"
          >
            {{ reactionLabel }}
            <span class="text-bold">
              {{ originalMewAuthor.fields[PROFILE_FIELDS.DISPLAY_NAME] }}
            </span>
            @{{ originalMewAuthor.nickname }}
          </router-link>
        </span>

        <q-space />

        <span class="q-ml-md text-caption">
          <Timestamp :timestamp="feedMew.action.timestamp" />
        </span>
      </div>

      <MewContent
        :feed-mew="isMewMew && originalMew ? originalMew : feedMew"
        class="q-my-sm cursor-pointer"
        @click="onMewClick(feedMew)"
      />

      <div>
        <q-btn :disable="isUpdatingLick" size="sm" flat @click="toggleLickMew">
          <q-icon
            name="svguse:/icons.svg#lick"
            :color="isLickedByMe ? 'pink-4' : 'transparent'"
            class="q-mr-xs"
          />
          {{ feedMew.licks.length }}
          <q-tooltip :delay="TOOLTIP_DELAY">Lick mew</q-tooltip>
        </q-btn>
        <q-btn size="sm" icon="reply" flat @click="replyToMew">
          {{ feedMew.replies.length }}
          <q-tooltip :delay="TOOLTIP_DELAY">Reply to mew</q-tooltip>
        </q-btn>
        <q-btn size="sm" icon="forward" flat @click="mewMew">
          {{ feedMew.mewmews.length }}
          <q-tooltip :delay="TOOLTIP_DELAY">Mewmew mew</q-tooltip>
        </q-btn>
        <q-btn size="sm" icon="format_quote" flat @click="quote">
          {{ feedMew.quotes.length }}
          <q-tooltip :delay="TOOLTIP_DELAY">Quote mew</q-tooltip>
        </q-btn>
      </div>
    </q-item-section>
  </q-item>
</template>

<script setup lang="ts">
import { ROUTES } from "@/router";
import {
  createMew,
  getFeedMewAndContext,
  lickMew,
  unlickMew,
} from "@/services/clutter-dna";
import { useProfilesStore } from "@/services/profiles-store";
import {
  CreateMewInput,
  FeedMew,
  MewTypeName,
  PROFILE_FIELDS,
  TOOLTIP_DELAY,
} from "@/types/types";
import { isSameHash } from "@/utils/hash";
import { useProfileUtils } from "@/utils/profile";
import { Profile } from "@holochain-open-dev/profiles";
import { ActionHash, encodeHashToBase64 } from "@holochain/client";
import { useQuasar } from "quasar";
import { computed, onMounted, PropType, ref } from "vue";
import { useRouter } from "vue-router";
import AvatarWithPopup from "./AvatarWithPopup.vue";
import CreateMewDialog from "./CreateMewDialog.vue";
import MewContent from "./MewContent.vue";
import Timestamp from "./MewTimestamp.vue";

const props = defineProps({
  feedMew: { type: Object as PropType<FeedMew>, required: true },
  onPublishMew: {
    type: Function as PropType<() => Promise<void>>,
    required: true,
  },
  onToggleLickMew: {
    type: Function as PropType<(hash: ActionHash) => Promise<void>>,
    required: true,
  },
  contentInsetLevel: { type: Number, default: undefined },
});

const $q = useQuasar();

const router = useRouter();

const profilesStore = useProfilesStore();
const { isCurrentProfile, onAgentClick } = useProfileUtils();
const agentProfile = ref<Profile>();
const myAgentPubKey = profilesStore.value.client.client.myPubKey;

const isMewMew = computed(
  () => MewTypeName.MewMew in props.feedMew.mew.mewType
);
const isOriginal = computed(
  () => MewTypeName.Original in props.feedMew.mew.mewType
);
const isReply = computed(() => MewTypeName.Reply in props.feedMew.mew.mewType);

const originalMewHash =
  MewTypeName.MewMew in props.feedMew.mew.mewType
    ? props.feedMew.mew.mewType.mewMew
    : MewTypeName.Reply in props.feedMew.mew.mewType
    ? props.feedMew.mew.mewType.reply
    : MewTypeName.Quote in props.feedMew.mew.mewType
    ? props.feedMew.mew.mewType.quote
    : props.feedMew.mew.mewType.original;
const originalMew = ref<FeedMew>();
const originalMewAuthor = ref<Profile>();
const loadingOriginalMewAuthor = ref<boolean>();
const reactionLabel = computed(() =>
  isMewMew.value ? "mewmewed from" : isReply.value ? "replied to" : "quoted"
);

const isUpdatingLick = ref(false);
const isLickedByMe = computed(() =>
  props.feedMew.licks.some((lick) => isSameHash(lick, myAgentPubKey))
);

onMounted(async () => {
  agentProfile.value = await profilesStore.value.client.getAgentProfile(
    props.feedMew.action.author
  );

  if (!originalMewHash) {
    return;
  }
  getFeedMewAndContext(originalMewHash).then((mew) => {
    originalMew.value = mew;
    // load original mew author
    loadingOriginalMewAuthor.value = true;
    profilesStore.value.client
      .getAgentProfile(mew.action.author)
      .then((profile) => (originalMewAuthor.value = profile))
      .finally(() => (loadingOriginalMewAuthor.value = false));
  });
});

const onMewClick = (mew: FeedMew) => {
  router.push({
    name: ROUTES.yarn,
    params: { hash: encodeHashToBase64(mew.actionHash) },
  });
};

const toggleLickMew = async () => {
  isUpdatingLick.value = true;
  if (isLickedByMe.value) {
    await unlickMew(props.feedMew.actionHash);
  } else {
    await lickMew(props.feedMew.actionHash);
  }
  await props.onToggleLickMew(props.feedMew.actionHash);
  isUpdatingLick.value = false;
};

const replyToMew = () =>
  $q.dialog({
    component: CreateMewDialog,
    componentProps: {
      mewType: { [MewTypeName.Reply]: props.feedMew.actionHash },
      onPublishMew: props.onPublishMew,
      originalMew: props.feedMew,
      originalAuthor: agentProfile.value,
    },
  });

const mewMew = async () => {
  const mew: CreateMewInput = {
    mewType: { mewMew: props.feedMew.actionHash },
    text: null,
  };
  await createMew(mew);
  props.onPublishMew();
};

const quote = () =>
  $q.dialog({
    component: CreateMewDialog,
    componentProps: {
      mewType: { [MewTypeName.Quote]: props.feedMew.actionHash },
      onPublishMew: props.onPublishMew,
      originalMew: props.feedMew,
      originalAuthor: agentProfile.value,
    },
  });
</script>
