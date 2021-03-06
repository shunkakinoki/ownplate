<template>
  <section class="section">
    <h1>{{$t("profile.title")}}</h1>
    <b-field class="m-t-8" :label="$t('profile.loginStatus')">
      <p>{{loginStatus}}</p>
    </b-field>
    <div v-if="!user">
      <div class="align-center m-t-24">
        <div
          class="op-button-small tertiary"
          @click.prevent="handleSignIn"
        >{{ $t("profile.signIn") }}</div>
      </div>
      <div class="align-center m-t-24">
        <router-link to="/admin/user/signin">
          <div class="op-button-small tertiary">{{ $t("profile.signInRestaurant") }}</div>
        </router-link>
      </div>
      <b-modal :active.sync="loginVisible" :width="640">
        <div class="card">
          <div class="card-content">
            <phone-login v-on:dismissed="handleDismissed" />
          </div>
        </div>
      </b-modal>
    </div>
    <div v-if="user && claims">
      <!--b-field class="m-t-8" :label="$t('profile.displayName')">
        <p>{{displayName}}</p>
      </b-field-->
      <div v-if="user.phoneNumber">
        <b-field class="m-t-8" :label="$t('profile.lineConnection')">
          <p>{{lineConnection}}</p>
        </b-field>
        <div v-if="isLineUser">
          <b-field class="m-t-8" :label="$t('profile.lineFriend')">
            <p>{{lineFriend}}</p>
          </b-field>
          <div v-if="isFriend===false" class="align-center">
            <b-button
              class="b-reset op-button-small"
              style="background:#18b900"
              tag="a"
              :href="friendLink"
            >
              <i class="fab fa-line c-text-white-full m-l-24 m-r-8" style="font-size:24px" />
              <span class="c-text-white-full m-r-24">
                {{
                $t("profile.friendLink")
                }}
              </span>
            </b-button>
          </div>
        </div>
        <div v-else>
          <div v-if="isLineEnabled" class="align-center">
            <b-button
              class="b-reset op-button-small"
              style="background:#18b900"
              tag="a"
              :href="lineAuth"
            >
              <i class="fab fa-line c-text-white-full m-l-24 m-r-8" style="font-size:24px" />
              <span class="c-text-white-full m-r-24">
                {{
                $t("line.notifyMe")
                }}
              </span>
            </b-button>
          </div>
        </div>
        <div class="align-center m-t-24">
          <router-link to="/u/history">
            <div class="op-button-small tertiary">{{ $t("order.history") }}</div>
          </router-link>
        </div>
      </div>
      <div class="align-center m-t-24">
        <div
          class="op-button-small tertiary"
          @click.prevent="handleSignOut"
        >{{ $t("menu.signOut") }}</div>
      </div>
      <div v-if="user.phoneNumber" class="align-center m-t-24">
        <b-button @click="handleDeleteAccount" class="b-reset op-button-text">
          <span class="c-status-red">{{ $t("profile.deleteAccount") }}</span>
        </b-button>
        <b-modal :active.sync="reLoginVisible" :width="640">
          <div class="card">
            <div class="card-content">
              <phone-login v-on:dismissed="continueDelete" :relogin="user.phoneNumber" />
            </div>
          </div>
        </b-modal>
      </div>
    </div>
    <b-loading :is-full-page="false" :active="isDeletingAccount" :can-cancel="true"></b-loading>
  </section>
</template>

<script>
import { parsePhoneNumber, formatNational } from "~/plugins/phoneutil.js";
import { db, auth, firestore, functions } from "~/plugins/firebase.js";
import { ownPlateConfig } from "@/config/project";
import PhoneLogin from "~/app/auth/PhoneLogin";

export default {
  components: {
    PhoneLogin
  },
  data() {
    return {
      loginVisible: false,
      reLoginVisible: false,
      isFriend: undefined,
      isDeletingAccount: false
    };
  },
  created() {
    if (this.isLineUser) {
      this.checkFriend();
    }
  },
  watch: {
    isWindowActive(newValue) {
      if (newValue && this.isLineUser && !this.isFriend) {
        this.isFriend = undefined;
        this.checkFriend();
      }
    },
    isLineUser(newValue) {
      if (this.isFriend === undefined) {
        this.checkFriend();
      }
    }
  },
  computed: {
    isWindowActive() {
      return this.$store.state.isWindowActive;
    },
    friendLink() {
      return ownPlateConfig.line.FRIEND_LINK;
    },
    lineAuth() {
      return this.lineAuthURL("/callback/line", location.pathname);
    },
    claims() {
      return this.$store.state.claims;
    },
    lineConnection() {
      return this.isLineUser
        ? this.$t("profile.status.hasLine")
        : this.$t("profile.status.noLine");
    },
    lineFriend() {
      if (this.isFriend === undefined) {
        return this.$t("profile.status.verifying");
      }
      return this.isFriend
        ? this.$t("profile.status.isFriend")
        : this.$t("profile.status.noFriend");
    },
    displayName() {
      return this.user?.displayName || this.$t("profile.undefined");
    },
    loginStatus() {
      if (this.user) {
        if (this.user.email) {
          const extra = this.$store.getters.isSuperAdmin ? "*admin" : "";
          return `${this.$t("profile.status.email")}: ${
            this.user.email
          } ${extra}`;
        } else if (this.user.phoneNumber) {
          const number = parsePhoneNumber(this.user.phoneNumber);
          return `${this.$t("profile.status.phone")}: ${formatNational(
            number
          )}`;
        } else if (this.user.uid.slice(0, 5) === "line:") {
          return this.$t("profile.status.line");
        }
        return this.$t("profile.status.unexpected");
      }
      return this.$t("profile.status.none");
    }
  },
  methods: {
    handleDeleteAccount() {
      this.$store.commit("setAlert", {
        code: "profile.reallyDeleteAccount",
        callback: async () => {
          this.reLoginVisible = true;
        }
      });
    },
    async continueDelete(result) {
      console.log(result);
      this.reLoginVisible = false;
      if (result) {
        this.isDeletingAccount = true;
        try {
          const accountDelete = functions.httpsCallable("accountDelete");
          const { data } = await accountDelete();
          console.log("deleteAccount", data);
          await this.user.delete();
          console.log("deleted");
        } catch (error) {
          console.error(error);
        } finally {
          this.isDeletingAccount = false;
        }
      }
    },
    handleSignIn() {
      this.loginVisible = true;
    },
    handleSignOut() {
      console.log("handleSignOut");
      auth.signOut();
    },
    handleDismissed() {
      console.log("handleDismissed");
      this.loginVisible = false;
    },
    async checkFriend() {
      const lineVerifyFriend = functions.httpsCallable("lineVerifyFriend");
      try {
        const { data } = await lineVerifyFriend({});
        this.isFriend = data.result;
      } catch (error) {
        console.error(error);
      }
    }
  }
};
</script>
