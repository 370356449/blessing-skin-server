<template>
    <section class="content">
        <vue-good-table
            :rows="plugins"
            :columns="columns"
            :search-options="tableOptions.search"
            :pagination-options="tableOptions.pagination"
            styleClass="vgt-table striped"
        >
            <template slot="table-row" slot-scope="props">
                <span v-if="props.column.field === 'title'">
                    <strong>{{ props.formattedRow[props.column.field] }}</strong>
                    <div>{{ props.row.name }}</div>
                </span>
                <span v-else-if="props.column.field === 'dependencies'">
                    <span
                        v-if="props.row.dependencies.requirements.length === 0"
                    ><i v-t="'admin.noDependencies'"></i></span>
                    <div v-else>
                        <span
                            v-for="(semver, dep) in props.row.dependencies.requirements"
                            :key="dep"
                            class="label"
                            :class="`bg-${dep in props.row.dependencies.unsatisfiedRequirements ? 'red' : 'green'}`"
                        >
                            {{ dep }}: {{ semver }}
                            <br>
                        </span>
                    </div>
                </span>
                <span v-else-if="props.column.field === 'operations'">
                    <template v-if="props.row.installed">
                        <button
                            v-if="props.row.update_available"
                            class="btn btn-success btn-sm"
                            @click="updatePlugin(props.row)"
                            :disabled="installing === props.row.name"
                        >
                            <template v-if="installing === props.row.name">
                                <i class="fas fa-spinner fa-spin"></i> {{ $t('admin.pluginUpdating') }}
                            </template>
                            <template v-else>
                                <i class="fas fa-sync-alt"></i> {{ $t('admin.updatePlugin') }}
                            </template>
                        </button>
                        <button v-else-if="props.row.enabled" class="btn btn-primary btn-sm" disabled>
                            <i class="fas fa-check"></i> {{ $t('admin.statusEnabled') }}
                        </button>
                        <button v-else class="btn btn-primary btn-sm" @click="enablePlugin(props.row)">
                            <i class="fas fa-plug"></i> {{ $t('admin.enablePlugin') }}
                        </button>
                    </template>
                    <button
                        v-else
                        class="btn btn-default btn-sm"
                        @click="installPlugin(props.row)"
                        :disabled="installing === props.row.name"
                    >
                        <template v-if="installing === props.row.name">
                            <i class="fas fa-spinner fa-spin"></i> {{ $t('admin.pluginInstalling') }}
                        </template>
                        <template v-else>
                            <i class="fas fa-download"></i> {{ $t('admin.installPlugin') }}
                        </template>
                    </button>
                </span>
                <span v-else v-text="props.formattedRow[props.column.field]" />
            </template>
        </vue-good-table>
    </section>
</template>

<script>
import { VueGoodTable } from 'vue-good-table';
import 'vue-good-table/dist/vue-good-table.min.css';
import toastr from 'toastr';
import { swal } from '../../js/notify';

export default {
    name: 'Market',
    components: {
        VueGoodTable
    },
    data() {
        return {
            plugins: [],
            columns: [
                { field: 'title', label: this.$t('admin.pluginTitle') },
                {
                    field: 'description',
                    label: this.$t('admin.pluginDescription'),
                    sortable: false,
                    width: '37%'
                },
                { field: 'author', label: this.$t('admin.pluginAuthor') },
                {
                    field: 'version',
                    label: this.$t('admin.pluginVersion'),
                    sortable: false,
                    globalSearchDisabled: true
                },
                {
                    field: 'dependencies',
                    label: this.$t('admin.pluginDependencies'),
                    sortable: false,
                    globalSearchDisabled: true
                },
                {
                    field: 'operations',
                    label: this.$t('admin.operationsTitle'),
                    sortable: false,
                    globalSearchDisabled: true
                },
            ],
            tableOptions: {
                search: {
                    enabled: true,
                    placeholder: this.$t('vendor.datatable.search')
                },
                pagination: {
                    enabled: true,
                    nextLabel: this.$t('vendor.datatable.next'),
                    prevLabel: this.$t('vendor.datatable.prev'),
                    rowsPerPageLabel: this.$t('vendor.datatable.rowsPerPage'),
                    allLabel: this.$t('vendor.datatable.all'),
                    ofLabel: this.$t('vendor.datatable.of')
                }
            },
            installing: '',
        };
    },
    beforeMount() {
        this.fetchData();
    },
    methods: {
        async fetchData() {
            this.plugins = await this.$http.get('/admin/plugins/market-data');
        },
        async installPlugin({ name, originalIndex }) {
            this.installing = name;

            const { errno, msg } = await this.$http.post(
                '/admin/plugins/market/download',
                { name }
            );
            if (errno === 0) {
                toastr.success(msg);
                this.plugins[originalIndex].update_available = false;
                this.plugins[originalIndex].installed = true;
            } else {
                swal({ type: 'warning', text: msg });
            }

            this.installing = '';
        },
        async updatePlugin(plugin) {
            const { dismiss } = await swal({
                text: this.$t('admin.confirmUpdate', { plugin: plugin.title, old: plugin.installed, new: plugin.version }),
                type: 'question',
                showCancelButton: true
            });
            if (dismiss) {
                return;
            }

            this.installPlugin(plugin);
        },
        async enablePlugin({ name, dependencies: { requirements }, originalIndex }) {
            if (requirements.length === 0) {
                const { dismiss } = await swal({
                    text: this.$t('admin.noDependenciesNotice'),
                    type: 'warning',
                    showCancelButton: true
                });
                if (dismiss) {
                    return;
                }
            }

            const { errno, msg, reason } = await this.$http.post(
                '/admin/plugins/manage',
                { action: 'enable', name }
            );
            if (errno === 0) {
                toastr.success(msg);
                this.$set(this.plugins[originalIndex], 'enabled', true);
            } else {
                const div = document.createElement('div');
                const p = document.createElement('p');
                p.textContent = msg;
                div.appendChild(p);
                const ul = document.createElement('ul');
                reason.forEach(item => {
                    const li = document.createElement('li');
                    li.textContent = item;
                    ul.appendChild(li);
                });
                div.appendChild(ul);
                swal({
                    type: 'warning',
                    html: div.innerHTML.replace(/`([\w-_]+)`/g, '<code>$1</code>')
                });
            }
        },
    }
};
</script>
