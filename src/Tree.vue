<template>
    <v-card flat tile width="250" min-height="380" class="d-flex flex-column folders-tree-card">
        <div class="grow scroll-x">
            <v-treeview
                :open="open"
                :active="active"
                :items="items"
                :search="filter"
                :load-children="readFolder"
                v-on:update:active="activeChanged"
                item-key="path"
                item-text="basename"
                dense
                activatable
                transition
                class="folders-tree"
            >
                <template v-slot:prepend="{ item, open }">
                    <v-icon
                        v-if="item.type === 'dir'"
                    >{{ open ? 'mdi-folder-open-outline' : 'mdi-folder-outline' }}</v-icon>
                    <v-icon v-else aria-label="File directory">{{ icons[item.extension.toLowerCase()] || icons['other'] }}</v-icon>
                </template>
                <template v-slot:label="{ item }">
                    {{item.basename}}
                    <v-btn
                        icon
                        v-if="item.type === 'dir'"
                        @click.stop="readFolder(item)"
                        class="ml-1"
                        aria-label="Refresh"
                    >
                        <v-icon class="pa-0 mdi-18px" color="grey lighten-1">mdi-refresh</v-icon>
                    </v-btn>
                </template>
            </v-treeview>
        </div>
        <v-divider></v-divider>
        <v-toolbar dense flat class="shrink">
            <v-text-field
                solo
                flat
                hide-details
                label="Filter"
                v-model="filter"
                prepend-inner-icon="mdi-filter-outline"
                class="ml-n3"
            ></v-text-field>
            <v-tooltip top>
                <template v-slot:activator="{ on }">
                    <v-btn icon @click="init" v-on="on" aria-label="Collapse">
                        <v-icon>mdi-collapse-all-outline</v-icon>
                    </v-btn>
                </template>
                <span>Collapse All</span>
            </v-tooltip>
        </v-toolbar>
    </v-card>
</template>

<script>
export default {
    props: {
        icons: Object,
        storage: String,
        path: String,
        endpoints: Object,
        axios: Function,
        refreshPending: Boolean
    },
    data() {
        return {
            open: [],
            active: [],
            items: [],
            filter: ''
        };
    },
    methods: {
        init() {
            this.open = [];
            this.items = [];
            // set default files tree items (root item) in nextTick.
            // Otherwise this.open isn't cleared properly (due to syncing perhaps)
            setTimeout(() => {
                this.items = [
                    {
                        type: 'dir',
                        path: '/',
                        basename: 'root',
                        extension: '',
                        name: 'root',
                        children: []
                    }
                ];
            }, 0);
            if (this.path !== '') {
                this.$emit('path-changed', '');
            }
        },
        async readFolder(item) {
            this.$emit('loading', true);
            let url = this.endpoints.list.url
                .replace(new RegExp('{storage}', 'g'), this.storage)
                .replace(new RegExp('{path}', 'g'), item ? item.path : this.path);

            let config = {
                url,
                method: this.endpoints.list.method || 'get'
            };

            let response = await this.axios.request(config);
            this.$emit('items', response.data);

            // eslint-disable-next-line require-atomic-updates
            item.children = response.data.map(item => {
                if (item.type === 'dir') {
                    item.children = [];
                }
                return item;
            });

            this.$emit('loading', false);
        },
        activeChanged(active) {
            this.active = active;
            let path = '';
            if (active.length) {
                path = active[0];
            }
            if (this.path != path) {
                this.$emit('path-changed', path);
            }
        },
        findItem(path) {
            let stack = [];
            stack.push(this.items[0]);
            while (stack.length > 0) {
                let node = stack.pop();
                if (node.path == path) {
                    return node;
                } else if (node.children && node.children.length) {
                    for (let i = 0; i < node.children.length; i++) {
                        stack.push(node.children[i]);
                    }
                }
            }
            return null;
        }
    },
    watch: {
        storage() {
            this.init();
        },
        path() {
            this.active = [this.path];
            if (!this.open.includes(this.path)) {
                this.open.push(this.path);
            }
        },
        async refreshPending(val){
            if (val) {
                let item = this.findItem(this.path);
                await this.readFolder(item);
                this.$emit('refreshed');
            }
        }
    },
    created() {
        this.init();
    }
};
</script>

<style scoped>
.folders-tree-card {
    height: 100%;
}
.folders-tree-card .scroll-x {
    overflow-x: auto;
}

.folders-tree-card ::v-deep .folders-tree {
    width: fit-content;
    min-width: 250px;
}
.folders-tree-card ::v-deep .folders-tree >>> .v-treeview-node {
    cursor: pointer;
}
.folders-tree-card ::v-deep .folders-tree >>> .v-treeview-node:hover {
    background-color: rgba(0, 0, 0, 0.02);
}
</style>
