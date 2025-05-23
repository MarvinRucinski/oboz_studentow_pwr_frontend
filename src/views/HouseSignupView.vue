<script setup>
import TopBar from '../components/navigation/TopBar.vue'
import LoadingIndicator from '../components/LoadingIndicator.vue'

import { useApiDataStore } from '../stores/api.js'
import { mapStores } from 'pinia'

import HouseCard from '../components/HouseCard.vue'
import cryingIcon from '../assets/icons8-crying.png'

import { IonPage, IonContent } from '@ionic/vue';
</script>


<template>
    <ion-page>
        <ion-content :fullscreen="false">
            <main>
                <TopBar :title="'Zapisy' + (
                    apiDataStore.houseSignupsInfo.ready && apiDataStore.houseSignupsInfo.data.room_instead_of_house ? ' na pokoje' : ' na domki'
                )" backLink="/" />

                <div
                    v-if="apiDataStore.houses.ready && apiDataStore.profile.ready && apiDataStore.houseSignupsInfo.ready && apiDataStore.houseSignupsInfo.data.house_signups_active">
                    <div v-if="apiDataStore.profile.data[0].house && apiDataStore.houses.houseWithId(apiDataStore.profile.data[0].house.id)"
                        class="padding">

                        <h3>Twój pokój</h3>
                        <div class="my-house">
                            <RouterLink
                                :to="{ name: 'zapisy-na-domki-detail', params: { id: apiDataStore.profile.data[0].house.id } }">
                                <HouseCard :key="apiDataStore.profile.data[0].house.id"
                                    :house="apiDataStore.houses.houseWithId(apiDataStore.profile.data[0].house.id)" />
                            </RouterLink>
                        </div>

                        <h3 style="margin-top: 12px !important;">Pozostałe {{
                            apiDataStore.houseSignupsInfo.data.room_instead_of_house ? 'pokoje' : 'domki'
                        }}</h3>
                    </div>
                    <div v-if="apiDataStore.houses.ready && apiDataStore.profile.ready">
                        <TransitionGroup name="list" tag="div" class="houses" @before-leave="beforeLeave">
                            <RouterLink :to="{ name: 'zapisy-na-domki-detail', params: { id: house.id } }"
                                :key="house.id"
                                v-for="house in apiDataStore.houses.housesWithoutId(apiDataStore.profile.data[0].house ? apiDataStore.profile.data[0].house.id : null)">
                                <HouseCard :key="house.id" :house="house" />
                            </RouterLink>
                        </TransitionGroup>
                        <p class="no-free-houses"
                            v-if="!apiDataStore.houses.housesWithoutId(apiDataStore.profile.data[0].house ? apiDataStore.profile.data[0].house.id : null).length">
                            Brak wolnych {{ apiDataStore.houseSignupsInfo.data.room_instead_of_house ? 'pokoi' :
                                'domków'
                            }}</p>
                    </div>
                    <p v-if="apiDataStore.houses.error || apiDataStore.profile.error" class="error">{{
                        apiDataStore.houses.error ||
                        apiDataStore.profile.error }}</p>
                </div>

                <div class="padding info-screen"
                    v-else-if="apiDataStore.houseSignupsInfo.ready && !apiDataStore.houseSignupsInfo.data.house_signups_active">
                    <h3>Zapisy zamknięte!</h3>
                    <img :src="cryingIcon" alt="crying" style="width: 100px; margin: 20px auto; display: block;" />
                    <p>Przepraszamy, ale zapisy na {{ apiDataStore.houseSignupsInfo.data.room_instead_of_house ?
                        'pokoje' : 'domki'
                        }} są
                        zamknięte.</p>

                    <RouterLink to="/">
                        <button class="button" style="margin-top: 20px">Wyjdź</button>
                    </RouterLink>
                </div>

                <LoadingIndicator v-else />
            </main>
        </ion-content>
    </ion-page>
</template>

<script>
export default {
    data() {
        return {
            housesTimer: null,
        }
    },
    computed: {
        ...mapStores(useApiDataStore),
    },
    mounted() {
        this.apiDataStore.houseSignupsInfo.fetchData()
        this.apiDataStore.houses.fetchData()
        this.apiDataStore.profile.fetchData()
        this.apiDataStore.houseSignups.connect()

        this.housesTimer = setInterval(() => {
            this.apiDataStore.houses.fetchData()
        }, 1000 * 90)
    },
    beforeUnmount() {
        clearInterval(this.housesTimer)

        // disconnect from the websocket if not leaving for the house detail view
        if (this.$route.name !== 'zapisy-na-domki-detail') {
            this.apiDataStore.houseSignups.disconnect()
        }
    },
    methods: {
        beforeLeave(el) {
            const { marginLeft, marginTop, width, height } = window.getComputedStyle(el)

            el.style.left = `${el.offsetLeft - parseFloat(marginLeft, 10)}px`
            el.style.top = `${el.offsetTop - parseFloat(marginTop, 10)}px`
            el.style.width = width
            el.style.height = height
        }
    }
}
</script>

<style scoped>
.houses {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    gap: 20px;
    align-items: center;
    width: 100%;
    padding: 0 20px;
}

.houses>a {
    width: calc(50% - 10px);
}

.my-house {
    display: flex;
    flex-direction: row;
    align-items: center;
    width: 100%;
    justify-content: center;
}

.my-house>a {
    width: calc(70% - 10px);
}

.info-screen {
    text-align: center;
}

.info-screen h3 {
    font-size: 25px;
    margin-bottom: 20px;
}

.info-screen p {
    color: var(--text-gray);
}

.info-screen p a {
    color: var(--theme-dark)
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

    width: 160px;
    display: flex;
    justify-content: center;

    margin: 0 auto;
    margin-top: 20px;
}


.list-move,
/* apply transition to moving elements */
.list-enter-active,
.list-leave-active {
    transition: all .8s ease;
}

.list-enter-from,
.list-leave-to {
    opacity: 0;
    transform: translateY(-10px);
}

/* ensure leaving items are taken out of layout flow so that moving
   animations can be calculated correctly. */
.list-leave-active {
    position: absolute;
}

.no-free-houses {
    text-align: center;
    /* margin-top: 5px; */
}
</style>