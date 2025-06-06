<script setup>
import TopBar from '../components/navigation/TopBar.vue'
import LoadingIndicator from '../components/LoadingIndicator.vue'
import OverlayView from '../components/OverlayView.vue'

import PointsItemView from './PointsItemView.vue'

import { useApiDataStore } from '../stores/api.js'
import { mapStores } from 'pinia'

import { apiRequest } from '../stores/functions.js'
import { IonPage, IonContent } from '@ionic/vue';
</script>

<template>
    <ion-page>
        <ion-content :fullscreen="false">
            <main>
                <TopBar title="Punkty" backLink="/admin-menu" />

                <div class="padding-main" v-if="apiDataStore.points.ready">
                    <h3>Rodziaj grupy</h3>
                    <select v-model="selectedGroupType"
                        @input="event => { selectedGroup = ''; selectedPointType = ''; selectedGroupType = event.target.value }">
                        <option value="">Wszystkie</option>
                        <option v-for="groupType in apiDataStore.points.groupTypes" :key="groupType" :value="groupType">
                            {{ groupType }}
                        </option>
                    </select>

                    <div v-if="selectedGroupType != ''">
                        <h3>Kategoria punktów</h3>
                        <select v-model="selectedPointType">
                            <option value="">Wszystkie</option>
                            <option v-for="pointType in apiDataStore.points.pointTypes(selectedGroupType)"
                                :key="pointType" :value="pointType">
                                {{ pointType }}
                            </option>
                        </select>
                    </div>

                    <div class="filters">
                        <p>Filtry:</p>
                        <div class="filterOption" :class="{ filterOptionSelected: niezatwierdzoneFilter }"
                            @click="toggleNiezatwierdzoneFilter">Nie zatwierdzone</div>

                        <div class="filterOption" :class="{ filterOptionSelected: odrzuconeFilter }"
                            @click="toggleOdrzuconeFilter">Odrzucone</div>

                        <select v-model="selectedGroup" v-if="selectedGroupType != ''">
                            <option value="">Grupa</option>
                            <option v-for="group in apiDataStore.points.groups(selectedGroupType)" :key="group.id"
                                :value="group.id">
                                {{ group.name }}
                            </option>
                        </select>
                    </div>


                    <div style="margin-top: 20px;">
                        <div v-for="(data, index) in filterOdrzucone(filterNiezatwierdzone(apiDataStore.points.filtered(selectedGroupType, selectedPointType, selectedGroup)))"
                            :key="index">
                            <PointsItemView :points="data.numberOfPoints" :date="data.date" :validated="data.validated"
                                :rejected="data.rejected"
                                :points_type="data.type.name + ' (' + data.type.points_min + ' - ' + data.type.points_max + ' pkt)'"
                                :description="data.description" :group_name="data.group.name"
                                :added_by="data.addedBy.first_name + ' ' + data.addedBy.last_name"
                                @click="showPointsOverlay(index)" />

                            <OverlayView ref="pointsOverlay">
                                <div class="padding pointsOverlay">
                                    <PointsItemView style="width:100%" :points="data.numberOfPoints" :date="data.date"
                                        :validated="data.validated" :rejected="data.rejected"
                                        :validatedBy="data.validatedBy.first_name + ' ' + data.validatedBy.last_name"
                                        :validationDate="data.validationDate"
                                        :points_type="data.type.name + ' (' + data.type.points_min + ' - ' + data.type.points_max + ' pkt)'"
                                        :description="data.description" :group_name="data.group.name"
                                        :added_by="data.addedBy.first_name + ' ' + data.addedBy.last_name" />

                                    <div class="validation-buttons">
                                        <button class="button success" @click="validatePoints(data)"
                                            v-if="!data.validated && apiDataStore.permissions.ready && apiDataStore.permissions.hasPermission('can_validate_points') && !success && !loading">
                                            Zatwierdź
                                        </button>
                                        <button class="button error" @click="rejectPoints(data)"
                                            v-if="!data.rejected && apiDataStore.permissions.ready && apiDataStore.permissions.hasPermission('can_validate_points') && !success && !loading">
                                            Odrzuć
                                        </button>
                                    </div>
                                    <LoadingIndicator v-if="loading" inline small />
                                    <p>{{ error }}</p>

                                    <button class="button" @click="hidePointsOverlay(index)">Zamknij</button>
                                </div>
                            </OverlayView>
                        </div>
                    </div>
                </div>

                <LoadingIndicator v-if="apiDataStore.points.loading" />
                <p v-if="apiDataStore.points.error" class="error">{{ apiDataStore.points.error }}</p>
            </main>
        </ion-content>
    </ion-page>
</template>


<script>
export default {
    data() {
        return {
            timer: null,
            selectedGroupType: '',
            selectedPointType: '',
            selectedGroup: '',

            niezatwierdzoneFilter: false,
            odrzuconeFilter: false,

            loading: false,
            error: null,
            success: null,

            fetchOnClose: true
        }
    },
    computed: {
        ...mapStores(useApiDataStore),
    },
    mounted() {
        this.apiDataStore.points.fetchData()
        this.timer = setInterval(this.apiDataStore.points.fetchData, 100000)
    },
    methods: {
        toggleNiezatwierdzoneFilter() {
            this.niezatwierdzoneFilter = !this.niezatwierdzoneFilter
        },
        filterNiezatwierdzone(points) {
            if (this.niezatwierdzoneFilter) {
                return points.filter(point => !point.validated)
            } else {
                return points
            }
        },

        toggleOdrzuconeFilter() {
            this.odrzuconeFilter = !this.odrzuconeFilter
        },
        filterOdrzucone(points) {
            if (this.odrzuconeFilter) {
                return points.filter(point => point.rejected)
            } else {
                return points.filter(point => !point.rejected)
            }
        },

        showPointsOverlay(index) {
            this.error = null
            this.success = null
            this.$refs.pointsOverlay[index].show()
        },
        hidePointsOverlay(index) {
            if (this.fetchOnClose) {
                this.apiDataStore.points.fetchData()
            }
            this.$refs.pointsOverlay[index].hide()
        },
        validatePoints(data) {
            if (this.apiDataStore.permissions.ready && this.apiDataStore.permissions.hasPermission('can_validate_points')) {
                this.loading = true;
                apiRequest('../staff-api/validate-points/' + data.id + '/',
                    'PUT'
                )
                    .then((data) => {
                        this.success = data.success
                        this.error = data.error
                        if (this.success) {
                            this.error = 'Zatwierdzono'
                        }
                    })
                    .catch((error) => {
                        console.error('There was an error!', error)
                    })
                    .finally(() => {
                        this.loading = false
                        this.fetchOnClose = true
                    })
            }
        },
        rejectPoints(data) {
            if (this.apiDataStore.permissions.ready && this.apiDataStore.permissions.hasPermission('can_validate_points')) {
                this.loading = true;
                apiRequest('../staff-api/reject-points/' + data.id + '/',
                    'PUT'
                )
                    .then((data) => {
                        this.success = data.success
                        this.error = data.error
                        if (this.success) {
                            this.error = 'Odrzucono'
                        }
                    })
                    .catch((error) => {
                        console.error('There was an error!', error)
                    })
                    .finally(() => {
                        this.loading = false
                        this.fetchOnClose = true
                    })
            }
        }
    },
    beforeUnmount() {
        clearInterval(this.timer)
    }
}
</script>

<style scoped>
select {
    width: 100%;
    padding: 10px 35px 10px 15px;
    border-radius: 20px;
    border: 1px solid var(--text-gray);
    margin-bottom: 10px;
    font-size: 15px;

    border: none;
    outline: none;
    color: white;
    -webkit-appearance: none;
    -moz-appearance: none;
    appearance: none;

    background: url("data:image/svg+xml,%3Csvg width='16' height='16' viewBox='0 0 16 16' fill='none' xmlns='http://www.w3.org/2000/svg'%3E%3Cpath d='M4.66663 5.33337L8.00001 8.66671L11.3333 5.33337' stroke='white' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E%0A") no-repeat;
    background-position: calc(100% - 15px) center !important;
    background-color: var(--bg-light);
}

.filters {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 13px;
}

.filters p {
    color: var(--text-gray);
}

.filterOption {
    background: var(--bg-light);
    padding: 8px 15px;
    border-radius: 20px;

    color: white;
    cursor: pointer;
}

.filterOptionSelected {
    background: var(--radial-gradient);
}

.filters select {
    width: auto;
    margin: 0;
    font-size: 13px;
}

button {
    border-radius: 20px;
    border: none;
    color: white;
    padding: 10px 20px;
    font-size: 14px;
    line-height: 16px;
    cursor: pointer;

    background-color: var(--bg-light);

    width: 130px;
    display: flex;
    justify-content: center;

    margin-top: 20px;
}

button.success {
    background-color: green;
}

button.error {
    background-color: var(--red);
}

.pointsOverlay {
    margin: 0;
    margin-top: 30px;
    display: flex;
    flex-direction: column;
    align-items: center;
}

.validation-buttons {
    display: flex;
    gap: 10px;
}
</style>