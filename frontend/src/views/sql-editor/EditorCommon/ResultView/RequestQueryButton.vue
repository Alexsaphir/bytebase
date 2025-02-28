<template>
  <div v-if="available">
    <NButton text type="primary" :size="size" @click="onClick">
      {{ $t("sql-editor.request-query") }}
    </NButton>

    <GrantRequestPanel
      v-if="showPanel"
      :project-name="database.project"
      :database="database"
      :placement="'right'"
      :role="PresetRoleType.PROJECT_QUERIER"
      @close="showPanel = false"
    />
  </div>
</template>

<script setup lang="ts">
import { NButton } from "naive-ui";
import { computed, ref } from "vue";
import GrantRequestPanel from "@/components/GrantRequestPanel";
import {
  isValidDatabaseName,
  PresetRoleType,
  type ComposedDatabase,
} from "@/types";
import { hasPermissionToCreateRequestGrantIssue } from "@/utils";

const props = withDefaults(
  defineProps<{
    database: ComposedDatabase;
    size?: "tiny" | "medium";
  }>(),
  {
    size: "medium",
  }
);

const showPanel = ref(false);

const available = computed(() => {
  if (!isValidDatabaseName(props.database.name)) {
    return false;
  }

  return hasPermissionToCreateRequestGrantIssue(props.database);
});

const onClick = (e: MouseEvent) => {
  e.stopPropagation();
  e.preventDefault();
  showPanel.value = true;
};
</script>
