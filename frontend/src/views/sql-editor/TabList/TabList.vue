<template>
  <div
    class="flex justify-between items-center box-border text-gray-500 text-sm border-b pr-2 gap-1"
  >
    <div
      class="relative flex flex-1 flex-nowrap overflow-hidden h-[36px] pt-0.5"
    >
      <Draggable
        id="tab-list"
        ref="tabListRef"
        v-model="filteredTabIdList"
        item-key="id"
        animation="300"
        class="tab-list hide-scrollbar"
        :class="{
          'more-left': scrollState.moreLeft,
          'more-right': scrollState.moreRight,
        }"
        ghost-class="ghost"
        @start="state.dragging = true"
        @end="state.dragging = false"
        @scroll="recalculateScrollState"
      >
        <template
          #item="{ element: id, index }: { element: string; index: number }"
        >
          <TabItem
            :tab="tabStore.tabById(id)!"
            :index="index"
            :data-tab-id="id"
            @select="(tab) => handleSelectTab(tab)"
            @close="(tab, index) => handleRemoveTab(tab, index)"
            @contextmenu.stop.prevent="
              contextMenuRef?.show(tabStore.tabById(id)!, index, $event)
            "
          />
        </template>
      </Draggable>

      <button
        class="bg-gray-200/20 hover:bg-accent/10 ml-1 py-1 px-1.5 border-t border-x rounded-t hover:border-accent"
        @click="handleAddTab"
      >
        <PlusIcon class="h-5 w-5" stroke-width="2.5" />
      </button>
    </div>

    <div class="flex items-center gap-2">
      <SettingButton v-if="!hideSettingButton" size="small" />
      <BrandingLogoWrapper v-if="!hideProfile">
        <ProfileDropdown :link="true" />
      </BrandingLogoWrapper>
    </div>

    <ContextMenu ref="contextMenuRef" />
  </div>
</template>

<script lang="ts" setup>
import { useResizeObserver } from "@vueuse/core";
import { PlusIcon } from "lucide-vue-next";
import { useDialog } from "naive-ui";
import { storeToRefs } from "pinia";
import scrollIntoView from "scroll-into-view-if-needed";
import { ref, reactive, nextTick, computed, onMounted, watch } from "vue";
import { useI18n } from "vue-i18n";
import Draggable from "vuedraggable";
import ProfileDropdown from "@/components/ProfileDropdown.vue";
import { useEmitteryEventListener } from "@/composables/useEmitteryEventListener";
import {
  useAppFeature,
  useFilterStore,
  useSQLEditorStore,
  useSQLEditorTabStore,
} from "@/store";
import type { SQLEditorTab } from "@/types";
import {
  defer,
  emptySQLEditorConnection,
  suggestedTabTitleForSQLEditorConnection,
} from "@/utils";
import { SettingButton } from "../Setting";
import { useSheetContext } from "../Sheet";
import BrandingLogoWrapper from "./BrandingLogoWrapper.vue";
import ContextMenu from "./ContextMenu.vue";
import TabItem from "./TabItem";
import { provideTabListContext } from "./context";

type LocalState = {
  dragging: boolean;
  hoverTabId: string;
};

const tabStore = useSQLEditorTabStore();

const { t } = useI18n();
const { filter } = useFilterStore();
const dialog = useDialog();

const state = reactive<LocalState>({
  dragging: false,
  hoverTabId: "",
});
const hideProfile = useAppFeature("bb.feature.sql-editor.hide-profile");
const disableSetting = useAppFeature("bb.feature.sql-editor.disable-setting");
const { events: sheetEvents } = useSheetContext();
const tabListRef = ref<InstanceType<typeof Draggable>>();
const { strictProject } = storeToRefs(useSQLEditorStore());
const context = provideTabListContext();
const contextMenuRef = ref<InstanceType<typeof ContextMenu>>();

const scrollState = reactive({
  moreLeft: false,
  moreRight: false,
});

const hideSettingButton = computed(() => {
  if (disableSetting.value) {
    return true;
  }
  if (strictProject.value) {
    return true;
  }

  return false;
});

const filteredTabIdList = computed(() => {
  // If a database is selected, only show tabs that are associated with the database.
  if (filter.database) {
    return tabStore.tabIdList.filter((id) => {
      const tab = tabStore.tabById(id);
      if (
        !tab?.connection.database ||
        tab.connection.database === filter.database
      ) {
        return true;
      } else {
        return false;
      }
    });
  }

  return tabStore.tabIdList;
});

const handleSelectTab = async (tab: SQLEditorTab | undefined) => {
  tabStore.setCurrentTabId(tab?.id ?? "");
};

const handleAddTab = () => {
  const currentTab = tabStore.currentTab;
  const connection = currentTab?.connection
    ? { ...currentTab.connection }
    : emptySQLEditorConnection();
  const title = suggestedTabTitleForSQLEditorConnection(connection);
  tabStore.addTab({
    title,
    connection,
    // The newly created tab is "clean"
    status: "CLEAN",
  });
  nextTick(recalculateScrollState);
  sheetEvents.emit("add-sheet");
};

const handleRemoveTab = async (
  tab: SQLEditorTab,
  index: number,
  focusWhenConfirm = false
) => {
  const _defer = defer<boolean>();
  if (tab.mode === "WORKSHEET" && tab.status === "DIRTY") {
    if (focusWhenConfirm) {
      tabStore.setCurrentTabId(tab.id);
    }
    const $dialog = dialog.create({
      title: t("sql-editor.hint-tips.confirm-to-close-unsaved-sheet.title"),
      content: t("sql-editor.hint-tips.confirm-to-close-unsaved-sheet.content"),
      type: "warning",
      autoFocus: false,
      closable: false,
      maskClosable: false,
      closeOnEsc: false,
      onPositiveClick() {
        remove(index);
        $dialog.destroy();
        _defer.resolve(true);
      },
      onNegativeClick() {
        $dialog.destroy();
        _defer.resolve(false);
      },
      positiveText: t("sql-editor.close-sheet"),
      negativeText: t("common.cancel"),
      showIcon: false,
    });
  } else {
    remove(index);
    _defer.resolve(true);
  }

  function remove(index: number) {
    tabStore.removeTab(tab);

    // select a tab near the removed tab.
    const nextIndex = Math.min(index, tabStore.tabList.length - 1);
    const nextTab = tabStore.tabList[nextIndex];
    handleSelectTab(nextTab);

    nextTick(recalculateScrollState);
  }

  return _defer.promise;
};

const tabListElement = computed((): HTMLElement | undefined => {
  const list = tabListRef.value;
  if (!list) return undefined;
  const element = list.$el as HTMLElement;
  return element;
});

useResizeObserver(tabListRef, () => {
  recalculateScrollState();
});

const recalculateScrollState = () => {
  contextMenuRef.value?.hide();

  const element = tabListElement.value;
  if (!element) {
    return;
  }

  const { scrollWidth, offsetWidth, scrollLeft } = element;

  if (scrollLeft === 0) {
    scrollState.moreLeft = false;
  } else {
    scrollState.moreLeft = true;
  }

  if (scrollWidth > offsetWidth) {
    // The actual width is wider than the visible width
    const rightBound = scrollLeft + offsetWidth;
    if (rightBound < scrollWidth) {
      scrollState.moreRight = true;
    } else {
      scrollState.moreRight = false;
    }
  } else {
    scrollState.moreRight = false;
  }
};

watch(
  () => tabStore.currentTabId,
  (id) => {
    requestAnimationFrame(() => {
      // Scroll the selected tab into view if needed.
      const tabElements = Array.from(document.querySelectorAll(".tab-item"));
      const tabElement = tabElements.find(
        (elem) => elem.getAttribute("data-tab-id") === id
      );
      if (tabElement) {
        scrollIntoView(tabElement, {
          scrollMode: "if-needed",
        });
      }
    });
  },
  { immediate: true }
);

onMounted(() => recalculateScrollState());

useEmitteryEventListener(
  context.events,
  "close-tab",
  async ({ tab, index, action }) => {
    const { tabList } = tabStore;

    const remove = async (tab: SQLEditorTab, index: number) => {
      await handleRemoveTab(tab, index, true);
      await new Promise((r) => requestAnimationFrame(r));
    };

    if (action === "CLOSE") {
      await remove(tab, index);
      return;
    }
    const max = tabList.length - 1;
    if (action === "CLOSE_OTHERS") {
      for (let i = max; i > index; i--) {
        await remove(tabList[i], i);
      }
      for (let i = index - 1; i >= 0; i--) {
        await remove(tabList[i], i);
      }
      return;
    }
    if (action === "CLOSE_TO_THE_RIGHT") {
      for (let i = max; i > index; i--) {
        await remove(tabList[i], i);
      }
      return;
    }
    if (action === "CLOSE_SAVED") {
      for (let i = max; i >= 0; i--) {
        const tab = tabList[i];
        if (tab.status === "CLEAN") {
          await remove(tab, i);
        }
      }
      return;
    }
    if (action === "CLOSE_ALL") {
      for (let i = max; i >= 0; i--) {
        await remove(tabList[i], i);
      }
    }
  }
);
</script>

<style scoped lang="postcss">
.ghost {
  @apply opacity-50 bg-white;
}

.tab-list {
  @apply flex flex-nowrap overflow-x-auto max-w-full overscroll-none;
}
</style>
