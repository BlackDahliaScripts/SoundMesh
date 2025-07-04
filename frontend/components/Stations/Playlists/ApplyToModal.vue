<template>
    <modal
        id="apply_playlist_to_modal"
        ref="$modal"
        size="xl"
        centered
        :loading="loading"
        :title="$gettext('Apply Playlist to Folders')"
        @hidden="clearContents"
    >
        <div class="row g-3">
            <div class="col-md-4">
                <form-markup id="apply_to_playlist_name">
                    <template #label>
                        {{ $gettext('Playlist:') }}
                    </template>

                    {{ applyToResults.playlist.name }}
                </form-markup>
            </div>
            <div class="col-md-8">
                <form-group-checkbox
                    id="form_applyto_copy_playlist"
                    :field="v$.copyPlaylist"
                >
                    <template #label>
                        {{ $gettext('Create New Playlist for Each Folder') }}
                    </template>
                </form-group-checkbox>
            </div>
        </div>

        <div style="max-height: 300px; overflow-y: scroll">
            <data-table
                :fields="fields"
                :provider="itemProvider"
                :show-toolbar="false"
                selectable
                @row-selected="onRowSelected"
            />
        </div>

        <template #modal-footer>
            <button
                type="button"
                class="btn btn-secondary"
                @click="hide"
            >
                {{ $gettext('Close') }}
            </button>
            <button
                type="button"
                class="btn btn-primary"
                :disabled="selectedDirs.length === 0"
                @click="save"
            >
                {{ $gettext('Apply to Folders') }}
            </button>
        </template>
    </modal>
</template>

<script setup lang="ts">
import DataTable, {DataTableField} from "~/components/Common/DataTable.vue";
import {computed, ref, useTemplateRef} from "vue";
import {useTranslate} from "~/vendor/gettext";
import {useNotify} from "~/functions/useNotify";
import {useAxios} from "~/vendor/axios";
import FormMarkup from "~/components/Form/FormMarkup.vue";
import {useVuelidateOnForm} from "~/functions/useVuelidateOnForm";
import {map} from "lodash";
import {useResettableRef} from "~/functions/useResettableRef";
import FormGroupCheckbox from "~/components/Form/FormGroupCheckbox.vue";
import Modal from "~/components/Common/Modal.vue";
import {useHasModal} from "~/functions/useHasModal.ts";
import {HasRelistEmit} from "~/functions/useBaseEditModal.ts";
import {useClientItemProvider} from "~/functions/dataTable/useClientItemProvider.ts";

const emit = defineEmits<HasRelistEmit>();

const $modal = useTemplateRef('$modal');
const {show, hide} = useHasModal($modal);

const {$gettext} = useTranslate();

const fields: DataTableField[] = [
    {
        key: 'name',
        isRowHeader: true,
        label: $gettext('Directory')
    }
];

const loading = ref<boolean>(true);
const applyToUrl = ref<string | null>(null);

const {record: applyToResults, reset: resetApplyToResults} = useResettableRef({
    playlist: {
        id: null,
        name: ''
    },
    directories: [],
});

const itemProvider = useClientItemProvider(
    computed(() => applyToResults.value.directories)
);

const selectedDirs = ref([]);
const onRowSelected = (items) => {
    selectedDirs.value = map(items, 'path');
};

const {form, v$, resetForm, ifValid} = useVuelidateOnForm(
    {
        copyPlaylist: {}
    },
    {
        copyPlaylist: false
    }
);

const clearContents = () => {
    applyToUrl.value = null;
    selectedDirs.value = [];

    resetApplyToResults();
    resetForm();
};

const {axios} = useAxios();

const open = (newApplyToUrl: string) => {
    clearContents();

    applyToUrl.value = newApplyToUrl;
    loading.value = true;
    show();

    void axios.get(newApplyToUrl).then((resp) => {
        applyToResults.value = resp.data;
        loading.value = false;
    });
};

const {notifySuccess} = useNotify();

const save = () => {
    ifValid(() => {
        void axios.put(applyToUrl.value, {
            ...form.value,
            directories: selectedDirs.value
        }).then(() => {
            notifySuccess($gettext('Playlist successfully applied to folders.'));
        }).finally(() => {
            hide();
            emit('relist');
        });
    });
};

defineExpose({
    open
});
</script>
