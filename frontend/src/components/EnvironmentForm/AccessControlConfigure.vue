<template>
  <div class="flex flex-col gap-y-2">
    <div class="textlabel flex items-center space-x-2">
      <label>
        {{ $t("environment.access-control.title") }}
      </label>
      <FeatureBadge feature="bb.feature.access-control" />
    </div>
    <div class="w-full flex flex-col gap-4">
      <div class="w-full inline-flex items-center gap-x-2">
        <Switch
          :value="disableCopyDataPolicy"
          :text="true"
          :disabled="!allowUpdatePolicy"
          @update:value="updateDisableCopyDataPolicy"
        />
        <span class="textlabel">{{
          $t("environment.access-control.disable-copy-data-from-sql-editor")
        }}</span>
      </div>
      <div class="">
        <div class="w-full inline-flex items-center gap-x-2">
          <Switch
            :value="adminDataSourceQueruRestrictionEnabled"
            :text="true"
            :disabled="!allowUpdatePolicy"
            @update:value="switchDataSourceQueryPolicyEnabled"
          />
          <span class="textlabel">{{
            $t("environment.access-control.restrict-admin-data-sources.self")
          }}</span>
        </div>
        <div v-if="adminDataSourceQueruRestrictionEnabled" class="ml-12">
          <NRadioGroup
            :value="adminDataSourceQueruRestriction"
            :disabled="!allowUpdatePolicy"
            @update:value="updateAdminDataSourceQueryRestrctionPolicy"
          >
            <NRadio
              class="w-full"
              :value="DataSourceQueryPolicy_Restriction.DISALLOW"
            >
              {{
                $t(
                  "environment.access-control.restrict-admin-data-sources.disallow"
                )
              }}
            </NRadio>
            <NRadio
              class="w-full"
              :value="DataSourceQueryPolicy_Restriction.FALLBACK"
            >
              {{
                $t(
                  "environment.access-control.restrict-admin-data-sources.fallback"
                )
              }}
            </NRadio>
          </NRadioGroup>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { NRadioGroup, NRadio } from "naive-ui";
import { computed, watchEffect } from "vue";
import { useI18n } from "vue-i18n";
import { hasFeature, pushNotification, usePolicyV1Store } from "@/store";
import {
  DataSourceQueryPolicy_Restriction,
  PolicyType,
} from "@/types/proto/v1/org_policy_service";
import { hasWorkspacePermissionV2 } from "@/utils";
import { FeatureBadge } from "../FeatureGuard";
import { Switch } from "../v2";

const props = defineProps<{
  resource: string;
  allowEdit: boolean;
}>();

const policyStore = usePolicyV1Store();
const { t } = useI18n();

watchEffect(async () => {
  await Promise.all([
    policyStore.getOrFetchPolicyByParentAndType({
      parentPath: props.resource,
      policyType: PolicyType.DISABLE_COPY_DATA,
    }),
    policyStore.getOrFetchPolicyByParentAndType({
      parentPath: props.resource,
      policyType: PolicyType.DATA_SOURCE_QUERY,
    }),
  ]);
});

const disableCopyDataPolicy = computed(() => {
  return policyStore.getPolicyByParentAndType({
    parentPath: props.resource,
    policyType: PolicyType.DISABLE_COPY_DATA,
  })?.disableCopyDataPolicy?.active;
});

const dataSourceQueryPolicy = computed(() => {
  return policyStore.getPolicyByParentAndType({
    parentPath: props.resource,
    policyType: PolicyType.DATA_SOURCE_QUERY,
  })?.dataSourceQueryPolicy;
});

const adminDataSourceQueruRestriction = computed(() => {
  return dataSourceQueryPolicy.value?.adminDataSourceRestriction;
});

const adminDataSourceQueruRestrictionEnabled = computed(() => {
  return (
    adminDataSourceQueruRestriction.value &&
    [
      DataSourceQueryPolicy_Restriction.DISALLOW,
      DataSourceQueryPolicy_Restriction.FALLBACK,
    ].includes(adminDataSourceQueruRestriction.value)
  );
});

const allowUpdatePolicy = computed(() => {
  return (
    props.allowEdit &&
    hasWorkspacePermissionV2("bb.policies.update") &&
    hasFeature("bb.feature.access-control")
  );
});

const updateDisableCopyDataPolicy = async (on: boolean) => {
  await policyStore.createPolicy(props.resource, {
    type: PolicyType.DISABLE_COPY_DATA,
    disableCopyDataPolicy: {
      active: on,
    },
  });
  pushNotification({
    module: "bytebase",
    style: "SUCCESS",
    title: t("common.updated"),
  });
};

const switchDataSourceQueryPolicyEnabled = async (on: boolean) => {
  await updateAdminDataSourceQueryRestrctionPolicy(
    on
      ? DataSourceQueryPolicy_Restriction.DISALLOW
      : DataSourceQueryPolicy_Restriction.RESTRICTION_UNSPECIFIED
  );
};

const updateAdminDataSourceQueryRestrctionPolicy = async (
  restrction: DataSourceQueryPolicy_Restriction
) => {
  await policyStore.createPolicy(props.resource, {
    type: PolicyType.DATA_SOURCE_QUERY,
    dataSourceQueryPolicy: {
      adminDataSourceRestriction: restrction,
    },
  });
  pushNotification({
    module: "bytebase",
    style: "SUCCESS",
    title: t("common.updated"),
  });
};
</script>
