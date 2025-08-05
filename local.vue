<template>
  <AuthLayout>
    <div class="">hello</div>
    <div class="">rev2</div>
    <FilterMaintenanceModal
      v-model="isOpenSearchModal"
      :activeFilter="activeFilter"
      :activeFilterInfo="activeFilterInfo"
      :ACTIVE_FILTER="ACTIVE_FILTER"
      @search="onSearch"
      v-if="isOpenSearchModal"
    />
    <!-- Filter section -->
    <div class="sticky_title filters px-4">
      <div class="filter_header">
        {{ getTranslateText("HISTORY") }}
      </div>
      <div class="filter_options">
        <div class="filter_option">
          <BlockFilterMaintenance
            :label="getFilterLabel(filter)"
            :icon="filter.icon"
            @click="openSearchModal(filter.id)"
            v-for="filter in filters"
            :key="filter.id"
            :filter="filter"
          />
        </div>
        <button class="search_btn" @click="onSearch">
          <v-icon :icon="mdiMagnify" size="18" color="#fff" />
        </button>
      </div>
    </div>
    <div class="maintenance_history">
      <!-- Maintenance list -->
      <div class="maintenance_list">
        <template v-if="filteredData.length > 0">
          <div
            v-for="(item, key) in filteredData"
            :key="key"
            class="maintenance_card"
            @click="redirectToMaintenanceDetail(item.maintenance_id)"
          >
            <div class="card_header">
              <div class="date">
                <img src="/icon/history/calendar_blue.svg" alt="icon" />
                {{ formatTimeStampDate(item.regist_date) }}
              </div>
              <StateBadge
                :color="item.progress_color_code"
                :status="getProgressName(item.progress_id)"
              />
            </div>
            <div class="device_info">
              <div class="device_info_header">
                <div class="device_id">{{ item.maintenance_cd }}</div>
                <div class="device_type">
                  {{ item.sub_category_name }}
                </div>
              </div>

              <OrderTypeLabel
                :orderTypeId="item.order_type_id"
                :order_type_other_text="item.order_type_other_text"
              />
            </div>

            <div class="details">
              <ItemDetail title="MANUFACTURER" :value="item.manufacturer" />
              <ItemDetail title="MODEL" :value="item.model_number" />
              <ItemDetail title="APPLICANT" :value="item.requester_cd" />
            </div>
          </div>
        </template>
        <div class="no_data text-center" v-else>
          {{ getTranslateText("NO_DATA_AVAILABLE") }}
        </div>
      </div>
    </div>
  </AuthLayout>
</template>

<script setup lang="ts">
import OrderTypeLabel from "@/components/atoms/OrderTypeLabel.vue";
import StateBadge from "@/components/atoms/StateBadge.vue";
import BlockFilterMaintenance from "@/components/maintenance/BlockFilterMaintenance.vue";
import FilterMaintenanceModal from "@/components/maintenance/FilterMaintenanceModal.vue";
import ItemDetail from "@/components/maintenance/ItemDetail.vue";
import { PATH } from "@/constants/path";
import { NO_SELECTION_VALUE } from "@/constants/common";
import {
  filterDataByValue,
  formatTimeStampDate,
  getDateFilterOptions,
  getProgressName,
  getTranslateText,
  startLoading,
  stopLoading,
} from "@/helper/common";
import { filterDataByApplicationDate } from "@/helper/maintenanceHelper";
import AuthLayout from "@/layouts/AuthLayout.vue";
import { getPrevUrl } from "@/router";
import { useMaintenanceStore } from "@/stores/maintenanceStore";
import { useSubCategoryStore } from "@/stores/subCategoryStore";
import { IMaintenanceDetail } from "@/types/maintenance";
import _ from "lodash";
import { computed, onMounted, onUnmounted, ref, watch } from "vue";
import { useRoute, useRouter } from "vue-router";
import { mdiMagnify } from "@mdi/js";

const isOpenSearchModal = ref(false);
const activeFilter = ref("");
const openSearchModal = (filterId: string) => {
  isOpenSearchModal.value = true;
  activeFilter.value = filterId;
};
const ACTIVE_FILTER = {
  APPLICATION_DATE: "applicationDate",
  SUB_CATEGORY: "subCategory",
};

const title = computed(() => getTranslateText("HISTORY"));

const maintenanceStore = useMaintenanceStore();
const subCategoryStore = useSubCategoryStore();

const router = useRouter();
const redirectToMaintenanceDetail = (maintenanceId: number) => {
  router.push(`${PATH.MAINTENANCES}/${maintenanceId}`);
};

const filterDataBySearchConditions = (data: any[]) => {
  const { applicationDate, subCategory } = maintenanceStore.historyFilters;
  if (applicationDate) {
    data = filterDataByApplicationDate(data, applicationDate);
  }
  if (subCategory) {
    data = filterDataByValue(data, subCategory, "sub_category_id");
  }
  return data;
};

const historyList = computed(() => maintenanceStore.historyList);
let filteredData = ref<IMaintenanceDetail[]>([]);

onMounted(async () => {
  const prevUrl = getPrevUrl();
  //   reset history filters if not from maintenance detail page
  if (!new RegExp(`^${PATH.MAINTENANCES}/\\d+$`).test(prevUrl)) {
    maintenanceStore.resetHistoryFilters();
  }
  maintenanceStore.clearMaintenanceDetail();
  startLoading();
  await maintenanceStore.fetchHistoryList();
  filteredData.value = filterDataBySearchConditions(historyList.value);
  stopLoading();
});

const dateFilterOptions = computed(() => getDateFilterOptions());
const dateFilterOptionsObj = computed(() =>
  _.keyBy(dateFilterOptions.value, "value")
);

const subCategoriesObj = computed(() => subCategoryStore.subCategoriesObj);
const storeFilters = computed(() => maintenanceStore.historyFilters);

const filters = computed(() => [
  {
    id: ACTIVE_FILTER.SUB_CATEGORY,
    icon: "/icon/history/category.svg",
    label: "Sub category",
    value:
      subCategoriesObj.value[storeFilters.value.subCategory]?.sub_category_name,
    title: "FILTER_BY_SUB_CATEGORY",
    titleIcon: "/icon/search_maintenance/size_24/category.svg",
    selectLabel: getTranslateText("SUB_CATEGORY"),
  },
  {
    id: ACTIVE_FILTER.APPLICATION_DATE,
    icon: "/icon/history/calendar.svg",
    label: "Application date",
    value:
      dateFilterOptionsObj.value[storeFilters.value.applicationDate]?.label,
    title: "FILTER_BY_APPLICATION_DATE",
    titleIcon: "/icon/search_maintenance/size_24/calendar.svg",
    selectLabel: getTranslateText("DATE"),
  },
]);

const activeFilterInfo = computed(() => {
  return filters.value.find((filter) => filter.id === activeFilter.value);
});
const getFilterLabel = (filter: any) => {
  let filterLabel = "";
  if (filter.id === ACTIVE_FILTER.SUB_CATEGORY) {
    filterLabel =
      subCategoriesObj.value[storeFilters.value.subCategory]?.sub_category_name;
    if (storeFilters.value.subCategory === NO_SELECTION_VALUE) {
      filterLabel = getTranslateText("FILTER_NO_SELECTION");
    }
  } else if (filter.id === ACTIVE_FILTER.APPLICATION_DATE) {
    filterLabel =
      dateFilterOptionsObj.value[storeFilters.value.applicationDate]?.label;
  }
  return filterLabel || filter.selectLabel;
};
const onSearch = () => {
  filteredData.value = filterDataBySearchConditions(historyList.value);
};

watch(
  title,
  (newTitle) => {
    if (newTitle) {
      document.title = newTitle;
    }
  },
  { immediate: true }
);

onUnmounted(() => {
  if (storeFilters.value.subCategory === NO_SELECTION_VALUE) {
    maintenanceStore.setHistoryFilters("subCategory", "");
  }
});
</script>

<style scoped lang="scss">
.filters {
  padding-bottom: 16px;
}
.filter_header {
  font-weight: 600;
  font-size: 16px;
  line-height: 24px;
  text-align: center;
  color: #252a31;
  padding: 10px 0;
}
.filter_options {
  display: flex;
  gap: 8px;
  align-items: center;
}
.search_btn {
  background: #2291fb;
  width: 40px;
  height: 40px;
  padding: 8px;
  border-radius: 2px;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}
.filter_option {
  display: flex;
  align-items: center;
  gap: 8px;
  width: calc(100% - 48px);
}
.maintenance_history {
  padding: 0 16px 85px 16px;

  .maintenance_list {
    display: flex;
    gap: 16px;
    flex-direction: column;
  }

  .maintenance_card {
    padding: 8px 16px;
    border-radius: 4px;
    border: 1px solid #ededed;
    box-shadow: 1px 1px 10px 0px #0000001a;
    cursor: pointer;
  }

  .card_header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-bottom: 8px;
    border-bottom: 1px dashed #ededed;
  }

  .date {
    color: #2291fb;
    font-weight: 400;
    font-size: 13px;
    line-height: 20px;
    display: flex;
    align-items: center;
    gap: 4px;
    margin-right: 4px;
  }

  .device_info {
    margin-top: 8px;
    padding-bottom: 8px;
    border-bottom: 1px dashed #ededed;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 16px;
  }
  .device_info_header {
    flex: 1 0 60%;
  }

  .device_id {
    font-weight: 600;
    font-size: 16px;
    line-height: 20px;
    color: #262729;
  }

  .device_type {
    color: #8d9094;
    font-weight: 400;
    font-size: 12px;
    line-height: 20px;

    word-break: break-word;
    overflow: hidden;
    text-overflow: ellipsis;
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
  }

  .details {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 32px;
    margin-top: 8px;
  }
}
</style>
