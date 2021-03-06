<template>
    <v-container>
        <v-toolbar flat height="30px">
            <v-tooltip bottom class="first">
                <v-btn icon flat @click.stop="refresh" slot="activator" small>
                    <v-icon small>refresh</v-icon>
                </v-btn>
                <span>Refresh</span>
            </v-tooltip>
            
            <v-tooltip bottom>
                <v-btn icon flat @click.stop="openCreateProjectWindow" slot="activator" small>
                    <v-icon small>mdi-folder-plus</v-icon>
                </v-btn>
                <span>New Project</span>
            </v-tooltip>

            <v-tooltip bottom>
                <v-btn icon flat @click.stop="openCreateFileWindow" slot="activator" small>
                    <v-icon small>mdi-file-document</v-icon>
                </v-btn>
                <span>New File</span>
            </v-tooltip>

            <v-tooltip bottom>
                <v-btn icon flat @click.stop="packageProject" slot="activator" small>
                    <v-icon small>mdi-package-variant-closed</v-icon>
                </v-btn>
                <span>Package</span>
            </v-tooltip>

            <v-tooltip bottom>
                <v-btn icon flat @click.stop="openInExplorer" slot="activator" small>
                    <v-icon small>mdi-folder-multiple</v-icon>
                </v-btn>
                <span>Open In Explorer</span>
            </v-tooltip>

            <v-spacer></v-spacer>
            <v-tooltip bottom>
                <v-btn icon flat @click.stop="" slot="activator" small>
                    <v-icon small>more_vert</v-icon>
                </v-btn>
                <span>More...</span>
            </v-tooltip>
        </v-toolbar>
        <v-select
            ref="project_select"
            :items="project_items" 
            :value="selected" 
            :label="display_label" 
            solo 
            dense 
            :loading="loading" 
            :disabled="items.length <= 1"
            @input="getDirectory"
        ></v-select>
        <v-divider></v-divider>
        <file-displayer :files="clever_directory" :project="selected" class="file-displayer"></file-displayer>
        <v-divider></v-divider>
    </v-container>
</template>

<script>
    import { ipcRenderer, shell } from "electron";
    import FileDisplayer from "./explorer/FileDisplayer.vue";
    import CreateFileWindow from "../../../windows/CreateFile";
    import CreateProjectWindow from "../../../windows/CreateProject";
    import EventBus from '../../../scripts/EventBus';
    import TabSystem from '../../../scripts/TabSystem';
    import { BASE_PATH } from "../../../scripts/constants";
    import ZipFolder from "zip-a-folder";
    import fs from "fs";
    
    export default {
        name: "content-explorer",
        components: {
            FileDisplayer
        },
        created() {
            this.$root.$on("refreshExplorer", () => {
                this.refresh();
            });

            ipcRenderer.on("readDir", (event, args) => {
                this.directory = args.files;
                this.$store.commit("setExplorerFiles", args.files);
                this.$store.commit("loadAllPlugins", { args, selected: this.selected, base_path: this.base_path });
            });

            ipcRenderer.on("readProjects", (event, args) => {
                this.items = args.files;

                if (this.items.length == 0) {
                    this.display_label = "No projects found";
                } else if(this.selected == "") {
                    this.$set(this, "selected", this.findDefaultProject());
                }
            });

            this.getProjects({ event_name: "initialProjectLoad", func: () => {
                this.getDirectory();
            }});

            window.addEventListener("resize", this.on_resize);
        },
        destroyed() {
            this.listeners.forEach(e => {
                ipcRenderer.removeAllListeners(e);
            });
            this.$root.$off("refreshExplorer");

            window.removeEventListener("resize", this.on_resize);
        },
        data() {
            return {
                listeners: ["readDir", "readProjects"],
                items: [],
                directory: undefined,
                display_label: "Loading...",
                project_select_size: window.innerWidth / 7.5
            };
        },
        computed: {
            selected: {
                get() {
                    return this.$store.state.Explorer.project;
                },
                set(project) {
                    this.$store.commit("setExplorerProject", project);
                    EventBus.trigger("updateTabUI");
                    // EventBus.on("updateSelectedTab");
                }
            },
            loading() {
                return this.items.length == 0;
            },
            clever_directory() {
                return this.directory ? this.directory.children : [];
            },

            base_path() {
                return BASE_PATH;
            },
            project_items() {
                let size = Math.floor(this.project_select_size / 8.25);
                
                let tmp = [];
                this.items.forEach(e => tmp.push({ 
                    text: e.length > size && !e.includes(" ") ? e.substr(0, size) + "\u2026" : e, 
                    value: e 
                }));
                return tmp;
            }
        },
        methods: {
            refresh() {
                this.getProjects({
                    event_name: "refreshExplorer",
                    func: () => {
                        this.$store.commit("forceReloadNextPluginRequest");
                        console.log("[REFRESH] " + this.selected);
                        this.getDirectory();
                    }
                });
            },
            openCreateFileWindow() {
                new CreateFileWindow();
            },
            openCreateProjectWindow() {
                new CreateProjectWindow();
            },
            packageProject() {
                let project = this.$store.state.Explorer.project;
                let path = BASE_PATH + project;
                ZipFolder.zipFolder(path, `${path}.mcpack`, err => {
                    if(err) throw err;
                    fs.rename(`${path}.mcpack`, `${path}/${project}.mcpack`, (err) => {
                        if(err) throw err;
                        this.$root.$emit("refreshExplorer");
                    })
                });
            },
            openInExplorer() {
                shell.openExternal(BASE_PATH + this.$store.state.Explorer.project);
            },

            getProjects({ event_name, func }={}) {
                this.registerListener(event_name, func);

                ipcRenderer.send("getProjects", { 
                    event_name
                });
            },
            getDirectory(dir=this.selected, { event_name, func }={}) {
                if(dir != this.selected) this.$set(this, "selected", dir);
                TabSystem.select(0);
                this.registerListener(event_name, func);

                ipcRenderer.send("getDir", { 
                    path: dir, 
                    event_name
                });
            },

            registerListener(event_name, func) {
                if(event_name && !this.listeners.includes(event_name)) {
                    ipcRenderer.on(event_name, func);
                    this.listeners.push(event_name);
                }
            },
            on_resize() {
                this.project_select_size = window.innerWidth / 7.5;
            },

            findDefaultProject() {
                if(this.$store.state.Settings.default_project === undefined) return this.items[0];
                
                for(let i = 0; i < this.items.length; i++) {
                    if(this.items[i].toLowerCase() === this.$store.state.Settings.default_project.toLowerCase())
                        return this.items[i];
                }
                return this.items[0];
            }
        }
    }
</script>

<style scoped>
    div.container {
        padding: 0;
    }

    button {
        padding: 0;
        width: 16px;
        height: 28px;
    }
    .v-btn {
        margin: 0;
    }
    .first {
        padding-left: 0.1em;
    }
</style>

<style>
    nav .v-toolbar__content {
        padding: 0;
    }
</style>