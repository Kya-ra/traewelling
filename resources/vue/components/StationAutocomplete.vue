<script>
import FullScreenModal from "./FullScreenModal.vue";
import _ from "lodash";
import {trans} from "laravel-vue-i18n";
import VueDatePicker from '@vuepic/vue-datepicker';
import '@vuepic/vue-datepicker/dist/main.css'
import {DateTime} from "luxon";
import {useUserStore} from "../stores/user";
import AutocompleteListEntry from "./Checkin/AutocompleteListEntry.vue";

export default {
    setup() {
        const userStore = useUserStore();
        userStore.fetchSettings();

        return {userStore};
    },
    name: "StationAutocomplete",
    emits: ["update:station", "update:time", "update:travelType"],
    components: {AutocompleteListEntry, FullScreenModal, VueDatePicker},
    props: {
        station: {
            type: Object,
            required: false,
            default: null,
        },
        stationName: {
            type: String,
            required: false,
            default: null,
        },
        dashboard: {
            type: Boolean,
            required: false,
            default: false
        },
        time: {
            type: DateTime,
            required: false,
            default: null
        },
        showFilterButton: {
            type: Boolean,
            required: false,
            default: false
        },
        showGpsButton: {
            type: Boolean,
            required: false,
            default: false
        }
    },
    data() {
        return {
            recent: [],
            loading: false,
            autocompleteList: [],
            stationInput: "",
            showFilter: false,
            date: null,
            selectedStation: null,
            selectedType: null,
            fetchingGps: false,
            travelTypes: [
                {value: "express", color: "rgba(197,199,196,0.5)", icon: "fa-train", contrast: true},
                {value: "regional", color: "rgba(193,18,28,0.5)", icon: "fa-train"},
                {value: "suburban", color: "rgba(0,111,53,0.5)", icon: "fa-train", image: "/img/suburban.svg"},
                {value: "subway", color: "rgba(21,106,184,0.5)", icon: "fa-subway", image: "/img/subway.svg"},
                {value: "tram", color: "rgba(217,34,42,0.5)", icon: "fa-tram", image: "/img/tram.svg"},
                {value: "bus", color: "rgba(163,0,124,0.5)", icon: "fa-bus", image: "/img/bus.svg"},
                {value: "ferry", color: "rgba(21,106,184,0.5)", icon: "fa-ship"},
                {value: "taxi", color: "rgb(255,237,74,0.5)", icon: "fa-taxi", contrast: true},
            ]
        };
    },
    methods: {
        trans,
        showModal() {
            this.$refs.modal.show();
        },
        setHome() {
            if (!this.isHome) {
                this.userStore.setHome(this.station).catch((error) => {
                    console.error(error);
                    window.notyf.error(trans('action.error') + " (" + trans('action.set-home') + ")");
                })
            }
        },
        getRecent() {
            fetch(`/api/v1/trains/station/history`).then((response) => {
                response.json().then((result) => {
                    this.recent = result.data;
                });
            });
        },
        autocomplete() {
            this.loading = true;
            if (!this.stationInput || this.stationInput.length < 2) {
                this.autocompleteList = [];
                this.loading          = false;
                return;
            }
            let query = this.stationInput.replace(/%2F/, " ").replace(/\//, " ");
            fetch(`/api/v1/trains/station/autocomplete/${query}`).then((response) => {
                response.json().then((result) => {
                    this.autocompleteList = result.data;
                    this.loading          = false;
                });
            });
        },
        showPicker() {
            this.$refs.picker.openMenu();
        },
        setTime() {
            this.$emit("update:time", DateTime.fromJSDate(this.date).setZone('UTC').toISO());
        },
        setStationFromText() {
            this.setStation({name: this.stationInput});
        },
        setStation(item) {
            this.stationInput    = item.name;
            this.selectedStation = item;
            this.$emit("update:station", item);
            this.$refs.modal.hide();
            const url = `/stationboard?stationId=${item.id}&stationName=${item.name}`;
            if (this.$props.dashboard) {
                window.location = url;
            }
        },
        setTravelType(travelType) {
            this.selectedType = this.selectedType === travelType.value ? null : travelType.value;
            this.$emit("update:travelType", this.selectedType);
        },
        setStationFromGps() {
            this.fetchingGps = true;
            if (!navigator.geolocation) {
                this.fetchingGps = false;
                notyf.error(trans("stationboard.position-unavailable"));
                return;
            }

            navigator.geolocation.getCurrentPosition(
                (position) => {
                    window.location.href = `/trains/nearby?latitude=${position.coords.latitude}&longitude=${position.coords.longitude}`;
                },
                () => {
                    this.fetchingGps = false;
                    notyf.error(trans("stationboard.position-unavailable"));
                }
            );
        }
    },
    watch: {
        stationInput: _.debounce(function () {
            this.autocomplete();
        }, 500),
        stationName() {
            this.stationInput = this.stationName ? this.stationName : this.stationInput;
        },
        station() {
            this.selectedStation = this.station;
        }
    },
    mounted() {
        this.date            = this.time;
        this.stationInput    = this.stationName ? this.stationName : "";
        this.selectedStation = this.station;
        this.getRecent();
    },
    computed: {
        placeholder() {
            return `${trans('stationboard.station-placeholder')} ${trans('or-alternative')} ${trans('ril100')}`;
        },
        dark() {
            return localStorage.getItem('darkMode') === 'dark';
        },
        isHome() {
            return this.userStore.getHome && this.station && this.userStore.getHome.id === this.station.id;
        }
    }
}
</script>

<template>
    <FullScreenModal ref="modal">
        <template #header>
            <input type="text" name="station" class="form-control mobile-input-fs-16"
                   :placeholder="placeholder"
                   v-model="stationInput"
                   @keyup.enter="setStationFromText"
            />
        </template>
        <template #body>
            <ul class="list-group list-group-light list-group-small">
                <AutocompleteListEntry
                    v-show="autocompleteList.length === 0"
                    :text="trans('stationboard.search-by-location')"
                    prefix="fa fa-map-marker-alt"
                    @click="setStationFromGps"
                />
                <AutocompleteListEntry
                    v-show="autocompleteList.length === 0 && userStore.getHome"
                    :station="userStore.getHome"
                    prefix="fas fa-house"
                    @click="setStation(userStore.getHome)"
                />
                <AutocompleteListEntry
                    v-for="item in recent"
                    v-show="autocompleteList.length === 0"
                    :station="item"
                    @click="setStation(item)"
                />
                <AutocompleteListEntry
                    v-for="item in autocompleteList"
                    :station="item"
                    @click="setStation(item)"
                />
            </ul>
        </template>
    </FullScreenModal>
    <div class="card mb-4">
        <div class="card-header">
            {{ trans("stationboard.where-are-you") }}
            <a v-if="!dashboard && station" href="#" class="float-end" @click.prevent="setHome">
                <i @click="setHome" :class="{'fas': isHome, 'far': !isHome}" class="fa-star"></i>
            </a>
        </div>
        <div class="card-body">
            <div id="station-autocomplete-container" style="z-index: 3;">
                <div class="input-group mb-2 mr-sm-2">
                    <input type="text" name="station" class="form-control mobile-input-fs-16"
                           :placeholder="placeholder"
                           v-model="stationInput"
                           @focusin="showModal"
                           @keyup.enter="setStationFromText"
                    />
                    <button v-if="showFilterButton" type="button" class="btn btn-outline-dark stationSearchButton"
                            @click="showFilter = !showFilter">
                        <i class="fa fa-filter" aria-hidden="true"></i>
                    </button>
                    <button v-if="showGpsButton" type="button" class="btn btn-outline-dark stationSearchButton"
                            @click="setStationFromGps">
                        <i v-if="!fetchingGps" class="fa fa-map-marker-alt" aria-hidden="true"></i>
                        <div v-else class="spinner-border" role="status" style="height: 1rem; width: 1rem;">
                            <span class="visually-hidden">Loading...</span>
                        </div>
                    </button>
                    <button type="button" class="btn btn-outline-dark stationSearchButton" @click="showPicker">
                        <i class="fa fa-clock" aria-hidden="true"></i>
                    </button>
                </div>

                <div class="d-flex justify-content-center">
                    <Transition name="slide-fade">
                        <div class="flex-wrap" role="group" v-show="showFilter">
                            <button
                                v-for="travelType in travelTypes"
                                type="button"
                                class="btn btn-primary btn-sm btn-rounded text-center me-1"
                                :class="{'active': selectedType === travelType.value, 'better-contrast': travelType.contrast ?? false}"
                                value="travelType"
                                :style="{backgroundColor: travelType.color}"
                                @click="setTravelType(travelType)"
                            >
                                <img v-if="travelType.image" :src="travelType.image" alt="icon" class="product-icon">
                                <i v-else :class="`fa ${travelType.icon}`" aria-hidden="true"></i>
                            </button>
                        </div>
                    </Transition>
                </div>
                <VueDatePicker
                    v-model="date"
                    ref="picker"
                    @update:model-value="setTime"
                    time-picker-inline
                    :dark="dark"
                    :action-row="{ showSelect: true, showCancel: true, showNow: true, showPreview: true }"
                >
                    <template #trigger>
                        <button type="button" class="btn btn-outline-dark stationSearchButton" hidden>
                            <i class="fa fa-calendar" aria-hidden="true"></i>
                        </button>
                    </template>
                </VueDatePicker>
            </div>
        </div>
    </div>

</template>

<style lang="scss" scoped>
.autocomplete-item {
    background-color: var(--mdb-modal-bg) !important;
}

.slide-fade-leave-active,
.slide-fade-enter-active {
    transition: all 0.3s ease-out;
    overflow: hidden;
}

.slide-fade-enter-from,
.slide-fade-leave-to {
    transform: translateY(-20px);
    opacity: 0;
}

.product-icon {
    width: 1rem;
    height: 1rem;
    vertical-align: middle;
    display: inline;
}

.better-contrast {
    color: #4F4F4F;
}

.better-contrast:hover {
    color: #212529;
}

:root.dark {
    .better-contrast {
        color: #FFF;
    }

    .better-contrast:hover {
        color: #FFF;
    }
}
</style>
