<template>
  <div
    id="app"
    v-if="config"
    :class="[
      `theme-${config.theme}`,
      isDark ? 'is-dark' : 'is-light',
      !config.footer ? 'no-footer' : '',
    ]"
  >
    <DynamicTheme :themes="config.colors" />
    <div id="bighead">
      <section v-if="config.header" class="first-line">
        <div v-cloak class="container">
          <div class="header-wrapper">
            <div class="logo">
              <a href="#">
                <img
                  v-if="config.logo"
                  :src="config.logo"
                  alt="dashboard logo"
                />
              </a>
              <i v-if="config.icon" :class="config.icon"></i>
            </div>
            <div class="dashboard-title">
              <span class="headline">{{ config.subtitle }}</span>
              <h1>{{ config.title }}</h1>
            </div>
          </div>
        </div>
      </section>

      <Navbar
        :open="showMenu"
        :links="config.links"
        @navbar-toggle="showMenu = !showMenu"
      >
        <DarkMode
          @updated="isDark = $event"
          :colorSwitch="config.colorSwitch"
          v-show="!config.colorSwitch.hideButton"
        />

        <SettingToggle
          @updated="vlayout = $event"
          name="vlayout"
          icon="fa-list"
          iconAlt="fa-columns"
          :layoutSwitch="config.layoutSwitch"
          v-show="!config.layoutSwitch.hideButton"
        />

        <SearchInput
          class="navbar-item navbar-item-nobg is-inline-block-mobile"
          v-if="config.search"
          :hotkey="config.searchHotkey"
          @input="filterServices"
          @search-focus="showMenu = true"
          @search-open="navigateToFirstService"
          @search-cancel="filterServices"
        />
      </Navbar>
    </div>

    <section id="main-section">
      <div v-cloak class="container">
        <ConnectivityChecker
          v-if="config.connectivityCheck"
          @network-status-update="offline = $event"
        />
        <div>
          <!-- Optional messages -->
          <Message :item="config.message" />

          <!-- Horizontal layout -->
          <div
            v-if="!vlayout || filter"
            class="columns is-multiline is-variable is-1"
          >
            <template v-for="(group, groupIndex) in services">
              <h2
                v-if="group.name"
                class="column is-full group-title"
                :key="group.name"
              >
                <i v-if="group.icon" :class="['fa-fw', group.icon]"></i>
                <div v-else-if="group.logo" class="group-logo media-left">
                  <figure class="image is-48x48">
                    <img :src="group.logo" :alt="`${group.name} logo`" />
                  </figure>
                </div>
                {{ group.name }}
              </h2>
              <Service
                v-for="(item, index) in group.items"
                :key="'group-' + groupIndex + '-' + index"
                :item="item"
                :proxy="config.proxy"
                :class="['column', 'service', `is-${12 / config.columns}`]"
              />
            </template>
          </div>

          <!-- Vertical layout -->
          <div
            v-if="!filter && vlayout"
            class="columns is-multiline layout-vertical"
          >
            <div
              :class="['column', `is-${12 / config.columns}`]"
              v-for="(group, groupIndex) in services"
              :key="group.name"
            >
              <h2 v-if="group.name" class="group-title">
                <i v-if="group.icon" :class="['fa-fw', group.icon]"></i>
                <div v-else-if="group.logo" class="group-logo media-left">
                  <figure class="image is-48x48">
                    <img :src="group.logo" :alt="`${group.name} logo`" />
                  </figure>
                </div>
                {{ group.name }}
              </h2>
              <Service
                v-for="(item, index) in group.items"
                :key="'group-' + groupIndex + '-' + index"
                :item="item"
                :proxy="config.proxy"
                class="service"
              />
            </div>
          </div>
        </div>
      </div>
    </section>

    <footer class="footer">
      <div class="container">
        <div
          class="content has-text-centered"
          v-if="config.footer"
          v-html="config.footer"
        ></div>
      </div>
    </footer>
  </div>
</template>

<script>
const jsyaml = require("js-yaml");
const merge = require("lodash.merge");

import Navbar from "./components/Navbar.vue";
import ConnectivityChecker from "./components/ConnectivityChecker.vue";
import Service from "./components/Service.vue";
import Message from "./components/Message.vue";
import SearchInput from "./components/SearchInput.vue";
import SettingToggle from "./components/SettingToggle.vue";
import DarkMode from "./components/DarkMode.vue";
import DynamicTheme from "./components/DynamicTheme.vue";

import defaultConfig from "./defaults.yml";

export default {
  name: "App",
  components: {
    Navbar,
    ConnectivityChecker,
    Service,
    Message,
    SearchInput,
    SettingToggle,
    DarkMode,
    DynamicTheme,
  },
  data: function () {
    return {
      config: null,
      services: null,
      offline: false,
      filter: "",
      vlayout: true,
      isDark: null,
      showMenu: false,
    };
  },
  created: async function () {
    this.buildDashboard();
    window.onhashchange = this.buildDashboard;
  },
  methods: {
    buildDashboard: async function () {
      const defaults = jsyaml.load(defaultConfig);
      let config;
      try {
        config = await this.getConfig();
        const path =
          window.location.hash.substring(1) != ""
            ? window.location.hash.substring(1)
            : null;

        if (path) {
          let pathConfig = await this.getConfig(`assets/${path}.yml`); // the slash (/) is included in the pathname
          config = Object.assign(config, pathConfig);
        }
      } catch (error) {
        console.log(error);
        config = this.handleErrors("⚠️ Error loading configuration", error);
      }
      this.config = merge(defaults, config);
      this.services = this.config.services;
      document.title =
        this.config.documentTitle ||
        `${this.config.title} | ${this.config.subtitle}`;
      if (this.config.stylesheet) {
        let stylesheet = "";
        for (const file of this.config.stylesheet) {
          stylesheet += `@import "${file}";`;
        }
        this.createStylesheet(stylesheet);
      }
    },
    getConfig: function (path = "assets/config.yml") {
      return fetch(path).then((response) => {
        if (response.redirected) {
          // This allows to work with authentication proxies.
          window.location.href = response.url;
          return;
        }
        if (!response.ok) {
          throw Error(`${response.statusText}: ${response.body}`);
        }

        const that = this;
        return response
          .text()
          .then((body) => {
            return jsyaml.load(body);
          })
          .then(function (config) {
            if (config.externalConfig) {
              return that.getConfig(config.externalConfig);
            }
            return config;
          });
      });
    },
    matchesFilter: function (item) {
      return (
        item.name.toLowerCase().includes(this.filter) ||
        (item.subtitle && item.subtitle.toLowerCase().includes(this.filter)) ||
        (item.tag && item.tag.toLowerCase().includes(this.filter))
      );
    },
    navigateToFirstService: function (target) {
      try {
        const service = this.services[0].items[0];
        window.open(service.url, target || service.target || "_self");
      } catch (error) {
        console.warning("fail to open service");
      }
    },
    filterServices: function (filter) {
      this.filter = filter;

      if (!filter) {
        this.services = this.config.services;
        return;
      }

      const searchResultItems = [];
      for (const group of this.config.services) {
        for (const item of group.items) {
          if (this.matchesFilter(item)) {
            searchResultItems.push(item);
          }
        }
      }

      this.services = [
        {
          name: filter,
          icon: "fas fa-search",
          items: searchResultItems,
        },
      ];
    },
    handleErrors: function (title, content) {
      return {
        message: {
          title: title,
          style: "is-danger",
          content: content,
        },
      };
    },
    createStylesheet: function (css) {
      let style = document.createElement("style");
      style.appendChild(document.createTextNode(css));
      document.head.appendChild(style);
    },
  },
};
</script>
