<template>
 <div>
  <div
    v-if="!freeTrialOver"
    id="app"
    class="h-screen flex flex-col font-sans overflow-hidden antialiased"
    :dir="languageDirection"
    :language="language"
  >
    <WindowsTitleBar
      v-if="platform === 'Windows'"
      :db-path="dbPath"
      :company-name="companyName"
    />
    <!-- Main Contents -->
    <Desk
      class="flex-1"
      v-if="activeScreen === 'Desk'"
      @change-db-file="showDbSelector"
    />
    <DatabaseSelector
      v-if="activeScreen === 'DatabaseSelector'"
      @file-selected="fileSelected"
    />
    <SetupWizard
      v-if="activeScreen === 'SetupWizard'"
      @setup-complete="setupComplete"
      @setup-canceled="showDbSelector"
    />

    <!-- Render target for toasts -->
    <div
      id="toast-container"
      class="absolute bottom-0 flex flex-col items-end mb-3 pe-6"
      style="width: 100%"
    >
      <div id="toast-target" />
    </div>
  </div>
  <div v-else class="free-trial">
    <div>
      <img src="./assets/img/fairbooks-logo.svg" />
    </div>
    <hr class="horizontal-rule"/>
    <h1 class="free-title">Your Free Trial is Over !</h1>
    <h1 class="contact-org">Contact The Organization</h1>
    <div class="contact-info">
      <h1>Phone: +254715705161</h1>
      <h1>Email: fairbooks.info@gmail.com</h1>
    </div>
  </div>
 </div>
</template>

<script>
import { ConfigKeys } from 'fyo/core/types';
import { RTL_LANGUAGES } from 'fyo/utils/consts';
import { ModelNameEnum } from 'models/types';
import { systemLanguageRef } from 'src/utils/refs';
import { computed } from 'vue';
import WindowsTitleBar from './components/WindowsTitleBar.vue';
import { handleErrorWithDialog } from './errorHandling';
import { fyo } from './initFyo';
import DatabaseSelector from './pages/DatabaseSelector.vue';
import Desk from './pages/Desk.vue';
import SetupWizard from './pages/SetupWizard/SetupWizard.vue';
import setupInstance from './setup/setupInstance';
import './styles/index.css';
import { initializeInstance } from './utils/initialization';
import { checkForUpdates } from './utils/ipcCalls';
import { updateConfigFiles } from './utils/misc';
import { Search } from './utils/search';
import { setGlobalShortcuts } from './utils/shortcuts';
import { routeTo } from './utils/ui';
import { Shortcuts, useKeys } from './utils/vueUtils';

export default {
  name: 'App',
  setup() {
    return { keys: useKeys() };
  },
  data() {
    return {
      activeScreen: null,
      dbPath: '',
      companyName: '',
      searcher: null,
      shortcuts: null,
      daysSinceFirstUse: (new Date() - new Date(fyo.config.get("freeTrialStartDate"))) / (1000 * 60 * 60 * 24),
      freeTrialOver: false,
    };
  },
  provide() {
    return {
      languageDirection: computed(() => this.languageDirection),
      searcher: computed(() => this.searcher),
      shortcuts: computed(() => this.shortcuts),
      keys: computed(() => this.keys),
    };
  },
  components: {
    Desk,
    SetupWizard,
    DatabaseSelector,
    WindowsTitleBar,
  },
  async mounted() {
    const shortcuts = new Shortcuts(this.keys);
    this.shortcuts = shortcuts;
    const lastSelectedFilePath = fyo.config.get(
      ConfigKeys.LastSelectedFilePath,
      null
    );
    // performing condition if days since the first use is greater
     if(this.daysSinceFirstUse>60){
      this.freeTrialOver = true;
    }

    if (!lastSelectedFilePath) {
      return (this.activeScreen = 'DatabaseSelector');
    }

    try {
      await this.fileSelected(lastSelectedFilePath, false);
    } catch (err) {
      await handleErrorWithDialog(err, undefined, true, true);
      await this.showDbSelector();
    }

    setGlobalShortcuts(shortcuts);
  },
  computed: {
    language() {
      return systemLanguageRef.value;
    },
    languageDirection() {
      return RTL_LANGUAGES.includes(this.language) ? 'rtl' : 'ltr';
    },
  },
  methods: {
    async setDesk(filePath) {
      this.activeScreen = 'Desk';
      await this.setDeskRoute();
      await fyo.telemetry.start(true);
      await checkForUpdates(false);
      this.dbPath = filePath;
      this.companyName = await fyo.getValue(
        ModelNameEnum.AccountingSettings,
        'companyName'
      );
      await this.setSearcher();
      await updateConfigFiles(fyo);
    },
    async setSearcher() {
      this.searcher = new Search(fyo);
      await this.searcher.initializeKeywords();
    },
    async fileSelected(filePath, isNew) {
      fyo.config.set(ConfigKeys.LastSelectedFilePath, filePath);
      if (isNew) {
        this.activeScreen = 'SetupWizard';
        return;
      }
      await this.showSetupWizardOrDesk(filePath);
    },
    async setupComplete(setupWizardOptions) {
      const filePath = fyo.config.get(ConfigKeys.LastSelectedFilePath);
      await setupInstance(filePath, setupWizardOptions, fyo);
      await this.setDesk(filePath);
    },
    async showSetupWizardOrDesk(filePath) {
      const countryCode = await fyo.db.connectToDatabase(filePath);
      const setupComplete = await fyo.getValue(
        ModelNameEnum.AccountingSettings,
        'setupComplete'
      );

      if (!setupComplete) {
        this.activeScreen = 'SetupWizard';
        return;
      }

      await initializeInstance(filePath, false, countryCode, fyo);
      await this.setDesk(filePath);
    },
    async setDeskRoute() {
      const { onboardingComplete } = await fyo.doc.getDoc('GetStarted');
      const { hideGetStarted } = await fyo.doc.getDoc('SystemSettings');

      if (hideGetStarted || onboardingComplete) {
        routeTo('/');
      } else {
        routeTo('/get-started');
      }
    },
    async showDbSelector() {
      fyo.config.set('lastSelectedFilePath', null);
      fyo.telemetry.stop();
      await fyo.purgeCache();
      this.activeScreen = 'DatabaseSelector';
      this.dbPath = '';
      this.searcher = null;
      this.companyName = '';
    },
  },
};
</script>

<style scoped>
  .free-trial{
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    margin-top: 50px;
  }
 
  .free-title{
    font-size: 30px;
    font-weight: bold;
    margin-top: 30px;
  }
  .horizontal-rule {
    border-top: 1px solid rgb(104, 103, 103);
    width: 30%;
    margin: 20px auto;
  }
  .contact-org{
    /* margin-left: 50px; */
    font-size: 20px;
    font-weight: 560;
  }
  .contact-info{
    margin-top: 20px;
  }
</style>