<template>
  <div class="dashboard-desktop">
    <new-organization ref="newOrganization" />
    <new-project ref="newProject" :organization-id="selectedOrganizationId" />
    <projects-trashcan ref="projectsTrashcan" />
    <project-import
      v-if="canImportProject(organizationId)"
      :project-file="importedProjectFile"
      :organization-id="organizationId"
      :is-shown.sync="showProjectImport"
    />
    <div v-if="!isReady">
      <v-progress-linear indeterminate />
    </div>
    <div v-else>
      <div class="projects-title">
        <v-layout align-center>
          <v-flex grow>
            <v-btn
              v-if="organizationId"
              class="back-button"
              small
              icon
              :to="{ name: 'dashboard-page' }"
            >
              <v-icon>mdi-arrow-left</v-icon>
            </v-btn>

            <template v-if="!organizationId">
              {{
                $t("Organizations & Projects")
              }}
            </template>
            <template v-if="organizationId && organization">
              {{
                organization.name
              }}
            </template>
          </v-flex>
          <v-flex shrink>
            <v-menu bottom left class="menu">
              <template v-slot:activator="{ on }">
                <v-btn small icon v-on="on">
                  <v-icon>mdi-dots-vertical</v-icon>
                </v-btn>
              </template>
              <v-list>
                <template v-if="!organizationId">
                  <v-list-item @click="newProject()">
                    <v-list-item-action>
                      <v-icon>mdi-format-list-bulleted</v-icon>
                    </v-list-item-action>
                    <v-list-item-title>
                      {{
                        $t("New project")
                      }}
                    </v-list-item-title>
                  </v-list-item>
                  <v-list-item @click="newOrganization()">
                    <v-list-item-action>
                      <v-icon>mdi-domain</v-icon>
                    </v-list-item-action>
                    <v-list-item-title>
                      {{
                        $t("New organization")
                      }}
                    </v-list-item-title>
                  </v-list-item>
                  <v-divider />
                </template>
                <!-- Import project from zip -->
                <v-list-item v-if="canImportProject(organizationId)" @click="importProject">
                  <v-list-item-action>
                    <v-icon>mdi-file-import</v-icon>
                  </v-list-item-action>
                  <v-list-item-title>
                    <upload-button
                      ref="uploadProject"
                      :is-uploading="isUploading"
                      @on-upload="uploadProject"
                    >
                      <span>{{ $t("project.import.importProject") }}</span>
                    </upload-button>
                  </v-list-item-title>
                </v-list-item>
                <v-list-item @click="$refs.projectsTrashcan.open()">
                  <v-list-item-action>
                    <v-icon>mdi-delete</v-icon>
                  </v-list-item-action>
                  <v-list-item-title>{{ $t("Trashcan") }}</v-list-item-title>
                </v-list-item>
              </v-list>
            </v-menu>
          </v-flex>
        </v-layout>
      </div>
      <v-divider />
      <template v-if="projects.length == 0 && organizations.length == 0">
        <empty-state
          class="main-empty-state"
          :description="$t('noProjectYet')"
          illustration="project"
        >
          <v-btn text @click="newProject()">
            {{
              $t("Create new project")
            }}
          </v-btn>
          <v-btn class="primary" @click="newOrganization()">
            {{
              $t("Create new organization")
            }}
          </v-btn>
        </empty-state>
      </template>
      <div class="projects-wrapper">
        <template v-if="favorites.length > 0">
          <div class="header">
            <div class="header-title">
              {{ $t("Favorites") }}
            </div>
          </div>
          <v-container
            ref="projects"
            v-resize="onResizeProjects"
            fluid
            grid-list-xl
            class="projects"
          >
            <v-layout row wrap>
              <v-flex v-for="project in favorites" :key="project._id" :class="cardClass">
                <dashboard-project-card :project="project" :user="user" />
              </v-flex>
            </v-layout>
          </v-container>
        </template>

        <template v-if="individuals.length > 0">
          <div class="header">
            <div class="header-title">
              {{ $t("Individuals") }}
            </div>
            <div class="header-action">
              <v-btn text solo class="action-button" @click="newProject()">
                <v-icon left>
                  mdi-plus
                </v-icon>
                {{ $t("New project") }}
              </v-btn>
            </div>
          </div>
          <v-list two-line class="list">
            <template v-for="project in individuals">
              <dashboard-project-list :key="project._id" :project="project" :user="user" />
            </template>
          </v-list>
        </template>

        <template v-for="organization in organizations">
          <div :key="`header-${organization._id}`" class="header">
            <div class="header-title">
              <router-link
                v-if="!organizationId"
                class="link"
                :to="{
                  name: 'dashboard-organization-page',
                  params: { organizationId: organization._id }
                }"
              >
                {{ organization.name }}
              </router-link>
              <template v-if="organizationId">
                {{ organization.name }}
              </template>
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    icon
                    small
                    text
                    color="grey darken-1"
                    @click.stop="openOrganizationTimeline(organization._id)"
                    v-on="on"
                  >
                    <v-icon>mdi-chart-timeline-variant</v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("Timeline") }}</span>
              </v-tooltip>
              <v-tooltip v-if="canManageOrganization(organization)" top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    icon
                    small
                    text
                    color="grey darken-1"
                    @click.stop="openOrganizationSettings(organization._id)"
                    v-on="on"
                  >
                    <v-icon>mdi-settings</v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("Settings") }}</span>
              </v-tooltip>
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    v-if="canDeleteOrganization(organization)"
                    icon
                    small
                    text
                    color="grey darken-1"
                    @click.stop="deleteOrganization(organization)"
                    v-on="on"
                  >
                    <v-icon>mdi-delete</v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("Delete") }}</span>
              </v-tooltip>
            </div>
            <div class="header-action">
              <v-btn text solo class="action-button" @click="newProject(organization._id)">
                <v-icon left>
                  mdi-plus
                </v-icon>
                {{ $t("New project") }}
              </v-btn>
            </div>
          </div>
          <v-list
            v-if="projectsByOrganization(organization, dashboardFilter).length > 0"
            :key="`projects-${organization._id}`"
            two-line
            class="list"
          >
            <template
              v-for="project in projectsByOrganization(
                organization,
                dashboardFilter
              )"
            >
              <dashboard-project-list :key="project._id" :project="project" :user="user" />
            </template>
          </v-list>

          <empty-state
            v-if="
              dashboardFilter === '' &&
                projectsByOrganization(organization).length == 0
            "
            :key="`${organization._id}-empty`"
            small
            :description="`Aucun projet disponible`"
            illustration="project"
          >
            <v-btn class="primary" @click="newProject(organization._id)">
              {{
                $t("Create new project")
              }}
            </v-btn>
          </empty-state>
        </template>
      </div>
    </div>
  </div>
</template>

<script>
import DashboardProjectCard from "/imports/ui/dashboard/desktop/DashboardProjectCard";
import DashboardProjectList from "/imports/ui/dashboard/desktop/DashboardProjectList";
import { Projects } from "/imports/api/projects/projects.js";
import { Organizations } from "/imports/api/organizations/organizations.js";
import DatesMixin from "/imports/ui/mixins/DatesMixin.js";
import { mapState } from "vuex";
import UploadButton from "/imports/ui/widgets/UploadButton";
import ProjectImport from "/imports/ui/projects/ProjectImportExport/ProjectImport/ProjectImport";
import { Permissions } from "/imports/api/permissions/permissions";

export default {
  components: {
    DashboardProjectCard,
    DashboardProjectList,
    UploadButton,
    ProjectImport
  },
  mixins: [DatesMixin],
  props: {
    organizationId: {
      type: String,
      default: ""
    }
  },
  data() {
    return {
      user: null,
      selectedOrganizationId: "",
      cardClass: "card1",
      showProjectImport: false,
      importedProjectFile: null,
      isUploading: false
    };
  },
  computed: {
    ...mapState(["currentUser", "dashboardFilter"]),
    isReady() {
      return this.$subReady.allProjects
      && this.$subReady.organizations && this.currentUser;
    }
  },
  mounted() {
    this.$store.dispatch("setWindowTitle", this.$t("Dashboard"));
    Meteor.call("users.getEmailPreferences", (error, user) => {
      if (error) {
        this.$notifyError(error);
        return;
      }
      this.user = user;
    });
  },
  meteor: {
    $subscribe: {
      allProjects: function() {
        return ["", this.organizationId];
      },
      organizations: function() {
        return ["", this.organizationId];
      }
    },
    projects: {
      params() {
        return {
          name: this.dashboardFilter
        };
      },
      deep: false,
      update({ name }) {
        const query = {};
        if (name && name.length > 0) {
          query.name = { $regex: `.*${name}.*`, $options: "i" };
        }
        return Projects.find(query, {
          sort: { organizationId: 1, state: 1, name: 1 }
        });
      }
    },
    organizations() {
      return Organizations.find({}, { sort: { name: 1 } });
    },
    organization: {
      params() {
        return {
          organizationId: this.organizationId
        };
      },
      update({ organizationId }) {
        const organization = Organizations.findOne({ _id: organizationId });
        this.$store.dispatch("organization/setCurrentOrganization", organization);
        return organization;
      }
    },
    favorites: {
      params() {
        return {
          name: this.dashboardFilter
        };
      },
      deep: false,
      update({ name }) {
        const user = Meteor.user() || { profile: {} };
        let favorites = [];
        if (user && user.profile) {
          favorites = user.profile.favoriteProjects || [];
        }
        const query = { _id: { $in: favorites } };
        if (name && name.length > 0) {
          query.name = { $regex: `.*${name}.*`, $options: "i" };
        }
        return Projects.find(query, {
          sort: { organizationId: 1, state: 1, name: 1 }
        });
      }
    },
    individuals: {
      params() {
        return {
          name: this.dashboardFilter
        };
      },
      deep: false,
      update({ name }) {
        const organizationIds = [];
        Organizations.find({})
          .fetch()
          .forEach((organization) => {
            organizationIds.push(organization._id);
          });
        const query = { organizationId: { $nin: organizationIds } };
        if (name && name.length > 0) {
          query.name = { $regex: `.*${name}.*`, $options: "i" };
        }
        return Projects.find(query, { sort: { state: 1, name: 1 } });
      }
    }
  },
  methods: {
    canImportProject(organizationId) {
      return Permissions.isAdmin(Meteor.userId(), organizationId)
      || Permissions.isAdmin(Meteor.userId());
    },
    importProject() {
      this.$refs.uploadProject.beginUpload();
    },
    uploadProject(file) {
      if (!file) return;
      this.importedProjectFile = file;
      this.showProjectImport = true;
    },
    newOrganization() {
      this.$refs.newOrganization.open();
    },
    toggleTaskAssignedTo() {
      this.currentUser.emailSettings.tasks.assignTo = !this.currentUser
        .emailSettings.tasks.assignTo;
      Meteor.call(
        "users.updateEmailPreferences",
        this.currentUser.emailSettings
      );
    },
    newProject(organizationId) {
      this.selectedOrganizationId = organizationId;
      this.$refs.newProject.open();
    },
    projectsByOrganization(organization, name) {
      const query = { organizationId: organization._id };
      if (name && name.length > 0) {
        query.name = { $regex: `.*${name}.*`, $options: "i" };
      }
      return Projects.find(query, { sort: { name: 1 } }).fetch();
    },
    openOrganization(id) {
      this.$router.push({
        name: "dashboard-organization-page",
        params: { organizationId: id }
      });
    },
    openOrganizationTimeline(id) {
      this.$router.push({
        name: "projects-timeline",
        params: { organizationId: id }
      });
    },
    openOrganizationSettings(id) {
      this.$router.push({
        name: "organization-settings",
        params: { organizationId: id }
      });
    },
    canManageOrganization(organization) {
      if (
        Permissions.isAdmin(Meteor.userId(), organization._id)
        || Permissions.isAdmin(Meteor.userId())
      ) {
        return true;
      }
      return false;
    },
    canDeleteOrganization(organization) {
      if (
        Permissions.isAdmin(Meteor.userId(), organization._id)
        || Permissions.isAdmin(Meteor.userId())
      ) {
        return true;
      }
      return false;
    },
    deleteOrganization(organization) {
      this.$confirm(this.$t("Delete organization?"), {
        title: organization.name,
        cancelText: this.$t("Cancel"),
        confirmText: this.$t("Delete")
      }).then((res) => {
        if (res) {
          Meteor.call(
            "organizations.remove",
            {
              organizationId: organization._id
            },
            (error) => {
              if (error) {
                this.$notifyError(error);
                return;
              }
              this.$notify(this.$t("Organization deleted"));
            }
          );
        }
      });
    },

    onResizeProjects() {
      const { projects } = this.$refs;
      const width = projects.offsetWidth;
      if (width > 600) {
        this.cardClass = "card3";
      } else if (width <= 600 && width > 400) {
        this.cardClass = "card2";
      } else {
        this.cardClass = "card1";
      }
    }
  }
};
</script>

<style scoped>
.link {
  text-decoration: none;
  color: black;
}

.link:hover {
  text-decoration: underline;
}

.back-button {
  margin-right: 12px;
}

.dashboard-desktop {
  padding: 2rem;
  display: flex;
  flex-direction: column;
  position: relative;
  flex: 1;
}

.header {
  display: flex;
  flex-direction: row;
  align-items: center;
}

.header-title {
  flex: 1;
  font-size: 20px;
  font-weight: 300;
}

.header-action {
  flex: 0;
}

.action-button {
  text-transform: none;
  color: #777;
  font-size: 12px;
}

.projects {
  padding-left: 0;
  padding-right: 12px;
}

.projects-title {
  text-transform: uppercase;
  font-weight: bold;
  margin-bottom: 12px;
  letter-spacing: 0.08em;
  flex: 0;
}

.projects-wrapper {
  flex: 1;
  flex-direction: column;
  overflow-y: auto;
  overflow-x: auto;
}

.projects-wrapper > .header:first-child {
  margin-top: 24px;
}

.main-empty-state {
  margin-top: 24px;
}

.card3 {
  flex-basis: calc(100% / 3);
  flex-grow: 0;
  max-width: calc(100% / 3);
}
.card2 {
  flex-basis: calc(100% / 2);
  flex-grow: 0;
  max-width: calc(100% / 2);
}

.card1 {
  flex-basis: 100%;
  flex-grow: 0;
  max-width: 100%;
}

.list {
  border-radius: 4px;
  border: 1px solid #ddd;
  margin-bottom: 24px;
}
</style>

<style></style>
