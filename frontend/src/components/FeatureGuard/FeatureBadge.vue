<!-- This component is used when uses don't have available plan to access the feature, or the instance is missing required license. -->
<!-- Normally this should be a blocker to use the feature -->
<template>
  <div
    v-if="instanceMissingLicense"
    :class="['text-accent cursor-pointer', customClass]"
    @click="state.showInstanceAssignmentDrawer = true"
  >
    <NTooltip :show-arrow="true">
      <template #trigger>
        <heroicons-solid:lock-closed class="w-5 h-5" />
      </template>
      <span class="w-56 text-sm">
        {{ $t("subscription.instance-assignment.missing-license-attention") }}
      </span>
    </NTooltip>
  </div>
  <div v-else-if="!hasFeature" :class="['text-accent', customClass]">
    <NTooltip :show-arrow="true">
      <template #trigger>
        <router-link
          v-if="clickable"
          :to="autoSubscriptionRoute($router)"
          exact-active-class=""
        >
          <heroicons-solid:sparkles class="w-5 h-5" />
        </router-link>
        <span v-else>
          <heroicons-solid:sparkles class="w-5 h-5" />
        </span>
      </template>
      <span class="w-56 text-sm">
        {{
          $t("subscription.require-subscription", {
            requiredPlan: $t(
              `subscription.plan.${planTypeToString(
                subscriptionStore.getMinimumRequiredPlan(feature)
              )}.title`
            ),
          })
        }}
      </span>
    </NTooltip>
  </div>
  <InstanceAssignment
    v-if="instanceMissingLicense && canManageSubscription"
    :show="state.showInstanceAssignmentDrawer"
    @dismiss="state.showInstanceAssignmentDrawer = false"
  />
</template>

<script lang="ts" setup>
import { NTooltip } from "naive-ui";
import type { PropType } from "vue";
import { reactive, computed } from "vue";
import { useSubscriptionV1Store } from "@/store";
import type { FeatureType } from "@/types";
import { planTypeToString } from "@/types";
import type {
  Instance,
  InstanceResource,
} from "@/types/proto/v1/instance_service";
import { autoSubscriptionRoute, hasWorkspacePermissionV2 } from "@/utils";
import InstanceAssignment from "../InstanceAssignment.vue";

interface LocalState {
  showInstanceAssignmentDrawer: boolean;
}

const props = defineProps({
  feature: {
    required: true,
    type: String as PropType<FeatureType>,
  },
  instance: {
    type: Object as PropType<Instance | InstanceResource>,
    default: undefined,
  },
  customClass: {
    require: false,
    default: "",
    type: String,
  },
  clickable: {
    type: Boolean,
    default: false,
  },
});

const state = reactive<LocalState>({
  showInstanceAssignmentDrawer: false,
});

const subscriptionStore = useSubscriptionV1Store();

const hasFeature = computed(() => {
  return subscriptionStore.hasInstanceFeature(props.feature, props.instance);
});

const instanceMissingLicense = computed(() => {
  return subscriptionStore.instanceMissingLicense(
    props.feature,
    props.instance
  );
});

const canManageSubscription = computed((): boolean => {
  return hasWorkspacePermissionV2("bb.instances.update");
});
</script>
