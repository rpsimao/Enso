<template>

    <div class="container">
        <div class="box">
            <div class="columns is-multiline">
                <div class="column">
                    <div class="columns is-multiline">
                        <div class="column is-half">
                            <vue-select :options="locales"
                                v-model="selectedLocale"
                                @input="getLangFile()"
                                :placeholder="__('Choose language')">
                            </vue-select>
                        </div>
                        <div class="column is-half has-text-right animated fadeIn is-hidden-mobile"
                            v-if="selectedLocale">
                            <p class="has-padding-top-small">
                                <b>{{ keysCount }}</b> {{ __('keys found') }}
                            </p>
                        </div>
                        <div class="column animated fadeIn"
                            v-if="selectedLocale">
                            <div class="field">
                                <p class="control has-icons-right">
                                    <input type="search"
                                        class="input"
                                        v-focus
                                        v-select-on-focus
                                        :placeholder="__('Search')"
                                        v-model="query"
                                        @keyup.enter="isNewKey ? addKey() : focusIt(null)">
                                    <span class="icon is-small is-right">
                                        <fa icon="search"></fa>
                                    </span>
                                </p>
                            </div>
                        </div>
                    </div>
                </div>
                <div class="column">
                    <div class="columns is-mobile has-text-centered">
                        <div class="column">
                            <button class="button is-success is-fullwidth"
                                v-if="isNewKey"
                                @click="addKey()">
                                {{ __('Add Key') }}
                            </button>
                        </div>
                        <div class="column">
                            <button @click="saveLangFile()"
                                v-if="selectedLocale"
                                class="button is-success is-fullwidth"
                                :class="{ 'is-loading': loading }">
                                {{ __('Update') }}
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div class="box"
            v-if="selectedLocale">
            <div class="columns is-hidden-mobile has-shadow-bottom">
                <div class="column is-half has-text-centered">
                    <h6 class="title is-6">{{ __('Key') }}</h6>
                </div>
                <div class="column is-half has-text-centered">
                    <h6 class="title is-6">{{ __('Value') }}</h6>
                </div>
            </div>
            <div :style="styleObject">
                <div class="columns"
                    :key="index"
                    v-for="(key, index) in filteredKeys">
                    <div class="column is-half">
                        <input type="text" class="input" readonly :value="key">
                    </div>
                    <div class="column is-half">
                        <div class="field has-addons">
                            <p class="control is-expanded">
                                <input type="text"
                                    v-select-on-focus
                                    v-model="langFile[key]"
                                    :id="key"
                                    class="input"
                                    @keyup.enter="focusIt('search-input')">
                            </p>
                            <p class="control">
                                <a class="button is-outlined is-danger"
                                    @click="removeKey(key)">
                                    <span class="icon is-small">
                                        <fa icon="trash-alt"></fa>
                                    </span>
                                </a>
                            </p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

</template>

<script>

import { mapGetters, mapState } from 'vuex';
import fontawesome from '@fortawesome/fontawesome';
import { faSearch, faTrashAlt } from '@fortawesome/fontawesome-free-solid/shakable.es';
import VueSelect from '../../../components/enso/vueforms/VueSelect.vue';

fontawesome.library.add(faSearch, faTrashAlt);

export default {
    components: { VueSelect },

    data() {
        return {
            langFile: {},
            locales: {},
            selectedLocale: null,
            query: null,
            boxHeight: 0,
            loading: false,
        };
    },

    computed: {
        ...mapState('layout', ['isMobile']),
        ...mapGetters('locale', ['__']),
        styleObject() {
            return {
                'max-height': this.boxHeight,
                'overflow-y': 'auto',
                'overflow-x': 'hidden',
            };
        },
        langKeys() {
            return Object.keys(this.langFile);
        },
        sortedKeys() {
            return this.langKeys.sort((a, b) => {
                if (a.toLowerCase() < b.toLowerCase()) return -1;
                if (a.toLowerCase() > b.toLowerCase()) return 1;
                return 0;
            });
        },
        filteredKeys() {
            if (!this.query) {
                return this.sortedKeys;
            }

            const self = this;

            return this.langKeys.filter(key => key.toLowerCase()
                .indexOf(self.query.toLowerCase()) > -1);
        },
        isNewKey() {
            return this.selectedLocale &&
                this.query && this.filteredKeys.indexOf(this.query) === -1;
        },
        keysCount() {
            return this.langKeys.length;
        },
    },

    watch: {
        isMobile: {
            handler: 'setBoxHeight',
        },
    },

    created() {
        this.init();
        this.setBoxHeight();
    },

    methods: {
        init() {
            this.loading = true;

            axios.get(route('system.localisation.editTexts', [], false)).then(({ data }) => {
                this.loading = false;
                this.locales = data.locales;
            }).catch((error) => {
                this.loading = false;
                this.handleError(error);
            });
        },
        getLangFile() {
            if (!this.selectedLocale) {
                this.langFile = {};
                return;
            }

            this.loading = true;

            axios.get(route('system.localisation.getLangFile', this.selectedLocale, false)).then(({ data }) => {
                this.loading = false;
                this.langFile = data;
            }).catch((error) => {
                this.loading = false;
                this.handleError(error);
            });
        },
        saveLangFile() {
            this.loading = true;

            axios.patch(route('system.localisation.saveLangFile', this.selectedLocale, false).toString(), {
                langFile: this.langFile,
            }).then(({ data }) => {
                this.loading = false;
                this.$toastr.success(data.message);
            }).catch((error) => {
                this.loading = false;
                this.handleError(error);
            });
        },
        addKey() {
            this.$set(this.langFile, this.query, null);
            this.focusIt();
        },
        removeKey(key) {
            this.$delete(this.langFile, key);
        },
        focusIt(id = null) {
            id = id || this.query;

            this.$nextTick(() => {
                document.getElementById(id).focus();
            });
        },
        setBoxHeight() {
            this.boxHeight = document.body.clientHeight - (this.isMobile ? 420 : 388);
        },
    },
};

</script>

<style>

    .has-shadow-bottom {
        -webkit-box-shadow: 0px 3px 5px -4px lightgray;
        -moz-box-shadow: 0px 3px 5px -4px lightgray;
        box-shadow: 0px 3px 5px -4px lightgray;
    }

</style>
