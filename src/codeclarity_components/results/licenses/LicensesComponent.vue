<script lang="ts" setup>
import { ref, type Ref, watch } from 'vue';
import SearchBar from '@/base_components/SearchBar.vue';
import BoxLoader from '@/base_components/BoxLoader.vue';

// import moment from 'moment';
import LicenseComponent from './LicenseComponent.vue';
// Import stores
import { useUserStore } from '@/stores/user';
import { useAuthStore } from '@/stores/auth';
import type { License } from '@/codeclarity_components/results/licenses/License';
import PaginationComponent from '@/base_components/PaginationComponent.vue';
import { ResultsRepository } from '@/codeclarity_components/results/results.repository';
import UtilitiesSort from '@/base_components/UtilitiesSort.vue';
import { SortDirection } from '@/utils/api/PaginatedRequestOptions';
import UtilitiesFilters, {
    createNewFilterState,
    FilterType,
    type FilterState
} from '@/base_components/UtilitiesFilters.vue';
import ActiveFilterBar from '@/base_components/ActiveFilterBar.vue';
import { ProjectsSortInterface } from '@/codeclarity_components/projects/project.repository';

export interface Props {
    analysisID?: string;
    projectID?: string;
}
const props = withDefaults(defineProps<Props>(), {
    analysisID: '',
    projectID: ''
});

const render = ref(false);
const sortDirection = ref(SortDirection.DESC);
const sortKey = ref(ProjectsSortInterface.LICENSE_TYPE);
const pageLimitSelected = ref(10);
const selectionPageLimit = [5, 10, 20, 30, 40, 50, 75, 100];
const nmbEntriesShowing = ref(pageLimitSelected.value);
const matchingItemsCount = ref(0);
const nmbEntriesTotal = ref(0);
const pageNumber = ref(0);
const totalPages = ref(10);
const filterApplied = ref(false);
const searchKey = ref('');
const placeholder = 'Search by licenses';
// const licenses_used = ref([]);
const licensesUsed: Ref<Array<License>> = ref([]);

const sortByOptions = [
    { key: 'type', label: 'Type' },
    { key: 'dep_count', label: 'Nmb. of deps' },
    { key: 'id', label: 'License Id' }
];

// Store setup
const resultsRepository: ResultsRepository = new ResultsRepository();
const userStore = useUserStore();
const authStore = useAuthStore();

watch([pageNumber, pageLimitSelected, sortKey, sortDirection, searchKey], () => {
    init();
});

// Filters
const filterState: Ref<FilterState> = ref(
    createNewFilterState({
        ImportState: {
            name: 'Language',
            type: FilterType.RADIO,
            data: {
                js: {
                    title: 'JavaScript',
                    value: true
                }
            }
        }
    })
);

async function init() {
    if (!userStore.getDefaultOrg) {
        throw new Error('No default org selected');
    }
    if (!authStore.getToken) {
        throw new Error('No default org selected');
    }
    if (props.projectID == '' || props.analysisID == '') {
        return;
    }
    try {
        const res = await resultsRepository.getLicenses({
            orgId: userStore.getDefaultOrg.id,
            projectId: props.projectID,
            analysisId: props.analysisID,
            workspace: '.',
            bearerToken: authStore.getToken,
            pagination: {
                page: pageNumber.value,
                entries_per_page: pageLimitSelected.value
            },
            sort: {
                sortKey: sortKey.value,
                sortDirection: sortDirection.value
            },
            active_filters: '',
            search_key: searchKey.value
        });
        licensesUsed.value = res.data;
        pageNumber.value = res.page;
        pageLimitSelected.value = res.entries_per_page;
        nmbEntriesShowing.value = res.entry_count;
        matchingItemsCount.value = res.matching_count;
        nmbEntriesTotal.value = res.total_entries;
        totalPages.value = res.total_pages;
        render.value = true;
    } catch (error) {
        console.error('error', error);
    }
}

init();
</script>

<template>
    <div class="flex flex-col gap-y-2">
        <div class="flex gap-4">
            <SearchBar v-model:search-key="searchKey" :placeholder="placeholder" />
            <UtilitiesFilters v-model:filter-state="filterState"></UtilitiesFilters>
        </div>

        <!--------------------------------------------------------------------------->
        <!--                           Active Filters list                         -->
        <!--------------------------------------------------------------------------->

        <ActiveFilterBar v-model:filter-state="filterState"></ActiveFilterBar>

        <UtilitiesSort
            v-model:page-limit-selected="pageLimitSelected"
            v-model:sort-key="sortKey"
            v-model:sort-direction="sortDirection"
            :selection-page-limit="selectionPageLimit"
            :sort-options="sortByOptions"
            :showing="nmbEntriesShowing"
            :total="nmbEntriesTotal"
        >
        </UtilitiesSort>
        <div v-if="render" class="flex flex-col gap-y-8">
            <div v-for="(license, index) in licensesUsed" :key="index">
                <LicenseComponent
                    :key="license.id"
                    :license="license"
                    :last="false"
                    :analysis-i-d="analysisID"
                    :project-i-d="projectID"
                />
            </div>
        </div>
        <div v-if="matchingItemsCount == 0 && filterApplied && render" style="">
            <div style="text-align: center">No licenses match the filter</div>
        </div>
        <div v-if="matchingItemsCount == 0 && !filterApplied && render" style="">
            <div style="text-align: center">No licenses</div>
        </div>
        <div v-if="!render" style="display: flex; flex-direction: column; row-gap: 10px">
            <BoxLoader
                v-for="index in 4"
                :key="index"
                :dimensions="{ width: '100%', height: '100px' }"
            />
        </div>
        <div class="text-[#484848] font-normal flex justify-between mt-7">
            <div style="">Showing {{ nmbEntriesShowing }} out of {{ nmbEntriesTotal }} entries</div>
            <PaginationComponent
                v-model:page="pageNumber"
                v-model:nmb-entries-showing="pageLimitSelected"
                v-model:nmb-entries-total="nmbEntriesTotal"
                v-model:total-pages="totalPages"
            />
        </div>
    </div>
</template>
