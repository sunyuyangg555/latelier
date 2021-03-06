<template>
  <div class="organization-settings-manage-users">
    <select-user
      :active.sync="showSelectUserDialog"
      :is-admin="canManageOrganization(organization)"
      @select="onSelectUser"
    />
    <div class="elevation-1 users">
      <v-list v-if="currentUser && $subReady.usersInOrganization">
        <v-subheader>
          {{ $t("Members") }}
          <v-btn text icon @click="showSelectUserDialog = true">
            <v-icon>mdi-plus</v-icon>
          </v-btn>
        </v-subheader>
        <template v-for="user in organizationUsers">
          <v-list-item :key="user._id">
            <v-list-item-avatar :color="isOnline(user)">
              <author-avatar :user-id="user" />
            </v-list-item-avatar>

            <v-list-item-content>
              <v-list-item-title>{{ formatUser(user) }}</v-list-item-title>
            </v-list-item-content>

            <v-list-item-action
              v-if="
                canManageOrganization(organization) &&
                  !isAdmin(user, organization) &&
                  userId != user._id
              "
            >
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    icon
                    ripple
                    @click.stop="setAdmin(user, organization)"
                    v-on="on"
                  >
                    <v-icon color="grey">
                      mdi-security
                    </v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("Grant admin rights") }}</span>
              </v-tooltip>
            </v-list-item-action>

            <v-list-item-action
              v-if="
                canManageOrganization(organization) &&
                  isAdmin(user, organization) &&
                  userId != user._id
              "
            >
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    icon
                    ripple
                    @click.stop="removeAdmin(user, organization)"
                    v-on="on"
                  >
                    <v-icon color="red">
                      mdi-security
                    </v-icon>
                  </v-btn>
                </template>
                <span>{{ $t("Remove admin rights") }}</span>
              </v-tooltip>
            </v-list-item-action>

            <v-list-item-action>
              <v-btn icon ripple @click.stop="removeUser(user)">
                <v-icon>mdi-delete</v-icon>
              </v-btn>
            </v-list-item-action>
          </v-list-item>
        </template>
      </v-list>
    </div>
  </div>
</template>

<script>
import { Meteor } from "meteor/meteor";
import { Permissions } from "/imports/api/permissions/permissions";
import usersMixin from "/imports/ui/mixins/UsersMixin.js";
import { mapState } from "vuex";

export default {
  name: "OrganizationSettingsManageUsers",
  mixins: [usersMixin],
  props: {
    organization: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      showSelectUserDialog: false
    };
  },
  computed: {
    ...mapState(["currentUser"])
  },
  meteor: {
    $subscribe: {
      usersInOrganization() {
        return [this.organization._id];
      }
    },
    organizationUsers: {
      params() {
        return {
          organization: this.organization
        };
      },
      deep: false,
      update({ organization }) {
        if (organization) {
          const members = organization.members || [];
          return Meteor.users.find({ _id: { $in: members } });
        }
        return null;
      }
    },
    userId() {
      return Meteor.userId();
    }
  },
  methods: {
    onSelectUser(user) {
      Meteor.call("organizations.addMember", {
        organizationId: this.organization._id,
        userId: user._id
      });
    },

    removeUser(user) {
      Meteor.call("organizations.removeMember", {
        organizationId: this.organization._id,
        userId: user._id
      });
    },

    isAdmin(user, organization) {
      return (
        Permissions.isAdmin(user._id, organization._id)
        || Permissions.isAdmin(user._id)
      );
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

    setAdmin(user, organization) {
      Meteor.call(
        "organizations.setAdmin",
        {
          organizationId: organization._id,
          userId: user._id
        },
        (error) => {
          if (error) {
            this.$notifyError(error);
          }
        }
      );
    },

    removeAdmin(user, organization) {
      Meteor.call(
        "organizations.removeAdmin",
        {
          organizationId: organization._id,
          userId: user._id
        },
        (error) => {
          if (error) {
            this.$notifyError(error);
          }
        }
      );
    }
  }
};
</script>

<style scoped>
.manage-user {
  overflow-y: scroll;
}

.toolbar {
  background-color: #2d6293;
}

.online {
  background-color: red;
}

.users {
  margin-top: 12px;
}
</style>
