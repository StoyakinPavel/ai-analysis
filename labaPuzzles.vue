<template>
  <div
    v-qa:id="'widget-laba-puzzles'"
    v-view-event:[extraSettings.analytics.enableViewEvents].widget
    :class="[styles['labaPuzzles'], styles[activeLevel]]"
    :style="backgroundStyle"
  >
    <transition name="fade" mode="out-in" appear>
      <div
        v-if="loadingRatingData"
        key="loader"
        :class="styles['labaPuzzlesLoader']"
      >
        <DsLoader
          :class="styles['labaPuzzlesLoaderSpinner']"
          :test-info="{ automatizationId: 'loader' }"
          :color="'var(--bgActiveSelectInverted)'"
          :size="600"
        />
      </div>
    </transition>

    <div :class="styles['labaPuzzlesContainer']">
      <div :class="styles['labaPuzzlesHeader']">
        <div :class="styles['labaPuzzlesHeaderLeft']">
          <div :class="styles['labaPuzzlesHeaderLogo']">
            <DsImage
              :use-cdn="shouldUseResizer('https://cdn1.ozone.ru/s3/sigma-static-368114f4-db69-4e5a-a17a-a8b11da72dc9/07206ba7514e8f57.png')"
              :use-web-p="useWebP"
              :size="getScale(extraSettings.imgScale)"
              :src="'https://cdn1.ozone.ru/s3/sigma-static-368114f4-db69-4e5a-a17a-a8b11da72dc9/07206ba7514e8f57.png'"
              :max-width="120"
              alt="ozon"
            />
          </div>

          <Points :points="points" />
        </div>

        <div :class="styles['labaPuzzlesHeaderRight']">
          <Timer
            :time-seconds="timeSeconds"
            :is-start="isStartTimer"
            :is-reset="isResetTimer"
            @reset-timer="resetTimerHandler"
            @passed-seconds="passedSecondsHandler"
            @remaining-seconds="remainingSecondsHandler"
            @time-is-up="timeIsUpHandler"
          />

          <template v-if="buttonHome.link">
            <a
              v-bind="buttonHomeLinkProps"
              :class="styles['labaPuzzlesButtonHome']"
              @click="(e) => handleClickLink(e, buttonHome.link)"
            >
              <DsIcon v-bind="buttonHomeIconProps" />
            </a>
          </template>
        </div>
      </div>

      <div :class="styles['labaPuzzlesGame']">
        <div :class="styles['labaPuzzlesField']">
          <DsImage
            :use-cdn="shouldUseResizer(getLevelData.gridImgSrc)"
            :use-web-p="useWebP"
            :size="getScale(extraSettings.imgScale)"
            :src="getLevelData.gridImgSrc"
            :class="styles['labaPuzzlesFieldGrid']"
            :test-info="{ automatizationId: 'image' }"
          />

          <DsImage
            :use-cdn="shouldUseResizer(getLevelData.assembledImgSrc)"
            :use-web-p="useWebP"
            :size="getScale(extraSettings.imgScale)"
            :src="getLevelData.assembledImgSrc"
            :class="styles['labaPuzzlesFieldAssembledImage']"
            :test-info="{ automatizationId: 'image' }"
          />

          <div :class="styles['labaPuzzlesFieldElements']">
            <div
              v-for="item in getLevelData.items"
              :key="item.id"
              :class="[
                styles['labaPuzzlesFieldElement'],
                checkElementIsVisibleById(item.id) && 'visible',
              ]"
            >
              <button
                type="button"
                :class="styles['labaPuzzlesFieldElementButton']"
                @click="onClickPuzzlesFieldElementButtonHandler(item)"
              >
                <DsImage
                  :use-cdn="shouldUseResizer(item.imgSrc)"
                  :use-web-p="useWebP"
                  :size="getScale(extraSettings.imgScale)"
                  :src="item.imgSrc"
                  :class="styles['labaPuzzlesFieldElementImage']"
                  :test-info="{ automatizationId: 'image' }"
                />
              </button>
            </div>
          </div>
        </div>

        <div :class="styles['labaPuzzlesInteraction']">
          <div
            :class="[
              styles['labaPuzzlesElements'],
              selectedElement && 'has-selected',
            ]"
          >
            <div
              v-for="item in getLevelData.items"
              :key="item.id"
              :class="[
                styles['labaPuzzlesElement'],
                checkElementIsActiveById(item.id) && 'active',
                checkElementIsVisibleById(item.id) && 'hide',
              ]"
            >
              <button
                type="button"
                :class="styles['labaPuzzlesElementButton']"
                @click="onClickPuzzlesElementButtonHandler(item)"
              >
                <DsImage
                  :use-cdn="shouldUseResizer(item.previewImgSrc)"
                  :use-web-p="useWebP"
                  :size="getScale(extraSettings.imgScale)"
                  :src="item.previewImgSrc"
                  :class="[styles['labaPuzzlesElementImage'], item.cssClasses]"
                  :test-info="{ automatizationId: 'image' }"
                />
              </button>
            </div>
          </div>

          <div :class="styles['labaPuzzlesActions']">
            <DsButton
              id="warning-modal-close-button"
              v-click-event:[extraSettings.analytics.enableClickEvents].button
              v-bind="getButtonProps(gameButtons.nextPuzzle, 'actionPrimary')"
              :class="styles['labaPuzzlesActionsButton']"
              size="600"
              @click="onClickNextPuzzleHandler"
            />

            <DsButton
              id="warning-modal-close-button"
              v-click-event:[extraSettings.analytics.enableClickEvents].button
              v-bind="getButtonProps(gameButtons.complete, 'actionPrimary')"
              :class="styles['labaPuzzlesActionsButton']"
              size="600"
              @click="onClickCompleteHandler"
            />
          </div>
        </div>
      </div>
    </div>

    <CompletedModal
      :is-modal-open="isVisibleCompletedModal"
      v-bind="completedModalProps"
      @repeat-game="onClickNextPuzzleHandler"
    />

    <WarningModal
      :is-modal-open="isVisibleWarningModal"
      v-bind="warningModalProps"
      @close="onCloseWarningModalHandler"
    />

    <RatingModal
      :is-modal-open="isVisibleRatingModal"
      v-bind="ratingModalProps"
      @close="onCloseRatingModalHandler"
    />
  </div>
</template>

<script>
import { DsButton } from '@bx-fe/ui-kit-ds-button';
import { DsIcon } from '@bx-fe/ui-kit-ds-icon';
import { DsImage } from '@bx-fe/ui-kit-ds-image';
import { DsLoader } from '@bx-fe/ui-kit-ds-loader';
import { QaDirectives } from '@bx-fe/qa-data';

import { STORAGE_KEY } from '../../common/constants/laba-api';
import {
  RESPONSE_STATUS_CODE,
  RESPONSE_STATUS_SUCCESS,
} from '../../common/constants/network';
import {
  GAME_AVATARS,
  GAMES_CODES,
} from '../../common/constants/games';
import { storageHelper } from '../../common/helpers/storage-helper';
import { isComponentInSigmaPreview } from '../../common/helpers/location';
import { AnalyticsMixin } from '../../common/mixins/analytics/index';
import { AsyncIconMixin } from '../../common/mixins/icons/async-icon';
import { BackgroundMixin } from '../../common/mixins/background/index';
import { CustomButtonMixin } from '../../common/mixins/buttons/custom-themes';
import { HandleClickLinkMixin } from '../../common/mixins/handleClick/index';
import { WebpMixin } from '../../common/mixins/webp/index';
import ApiService from '../../common/services/laba-services';
import Timer from '../../common/components/timer/index.vue';
import Points from '../../common/components/points/index.vue';
import CompletedModal from '../../common/components/completedModal/index.vue';
import WarningModal from '../../common/components/warningModal/index.vue';
import RatingModal from '../../common/components/ratingModal/index.vue';
import {
  PUZZLES,
  PUZZLES_LEVELS,
  PUZZLES_GAME_PROCESS,
  RATING_RAW,
} from './constants';
import styles from './style.module.scss';

export default {
  name: 'LabaPuzzles',
  components: {
    DsButton,
    DsIcon,
    DsImage,
    DsLoader,
    Timer,
    Points,
    CompletedModal,
    WarningModal,
    RatingModal,
  },
  directives: {
    qa:
      process.browser && document.cookie.includes('x-o3-ui-tests=true')
        ? QaDirectives
        : {},
  },
  mixins: [
    AnalyticsMixin,
    AsyncIconMixin,
    BackgroundMixin,
    CustomButtonMixin,
    HandleClickLinkMixin,
    WebpMixin,
  ],
  props: {
    buttonHome: {
      default: () => ({
        link: '',
        iconKey: '',
        iconColor: '',
        backgroundColor: '',
      }),
      type: Object,
      description: 'Настройка кнопки «Домой»',
    },
    modalsSettings: {
      default: () => ({
        completedModal: {
          image: '',
          heading: '',
          description: '',
          buttonRepeat: {
            text: '',
            theme: '',
            backgroundColor: '',
            textColor: '',
          },
          buttonReturn: {
            text: '',
            link: '',
            theme: '',
            backgroundColor: '',
            textColor: '',
          },
        },
        warningModal: {
          heading: '',
          description: '',
          button: {
            text: '',
            link: '',
            theme: '',
            backgroundColor: '',
            textColor: '',
          },
        },
      }),
      type: Object,
      description: 'Настройка карточек',
    },
    timer: {
      default: '',
      type: String,
      description: 'Настройки таймера',
    },
    gameButtons: {
      default: () => ({
        nextPuzzle: {
          text: '',
          theme: '',
          backgroundColor: '',
          textColor: '',
        },
        complete: {
          text: '',
          theme: '',
          backgroundColor: '',
          textColor: '',
        },
      }),
      type: Object,
      description: 'Настройки кнопок на игровом поле',
    },
    extraSettings: {
      default: () => ({
        analytics: {
          enableViewEvents: false,
          enableClickEvents: false,
        },
        imgScale: '1',
        customBackground: {
          color: '',
          imageUrl: '',
          disableCompression: false,
          gradient: {},
        },
      }),
      type: Object,
      description: 'Дополнительные настройки',
    },
  },
  data() {
    return {
      PUZZLES,
      PUZZLES_LEVELS,
      PUZZLES_GAME_PROCESS,
      RATING_RAW,

      styles,
      icons: {},

      siteData: null,
      isVisibleCompletedModal: false,
      isVisibleWarningModal: false,
      isVisibleRatingModal: false,

      allLevels: [
        PUZZLES_LEVELS.easy,
        PUZZLES_LEVELS.medium,
        PUZZLES_LEVELS.hard,
      ],

      assembledLevels: [],
      assembledPuzzlesActiveLevel: [],

      activeLevel: PUZZLES_LEVELS.easy,
      activeLevelGameIndex: 0,

      selectedElement: null,

      points: 0,

      isStartedGame: false,
      gameProcess: PUZZLES_GAME_PROCESS.nextPuzzle,

      isStartTimer: false,
      isResetTimer: false,
      timeSeconds: Number(this.timer) || 300,
      passedSeconds: 0,
      remainingSeconds: Number(this.timer) || 300,

      loadingRatingData: false,
      ratingData: [],
    };
  },
  computed: {
    isSigmaPreview() {
      return isComponentInSigmaPreview();
    },
    buttonHomeLinkProps() {
      const { link } = this.buttonHome;

      return {
        disabled: this.isStartedGame,
        href: link,
        target: link && link.startsWith('http') ? '_blank' : '_self',
      };
    },
    buttonHomeIconProps() {
      const { iconKey, iconColor, backgroundColor } = this.buttonHome;

      return {
        size: '500',
        shape: 'circle',
        icon: this.icons[iconKey] || {},
        iconName: iconKey,
        iconColor: iconColor || 'var(--graphicTertiary)',
        backgroundColor,
      };
    },
    completedModalProps() {
      const baseProps = {
        image: 'https://cdn1.ozone.ru/s3/sigma-static-368114f4-db69-4e5a-a17a-a8b11da72dc9/20c12d1d0426bed0.jpg',
        heading: 'Молодец',
        description: '',
        buttonRepeat: {
          text: 'Следующая картинка',
          theme: 'accentPrimary',
          backgroundColor: 'accentPrimary',
          textColor: 'accentPrimary',
        },
        closeButton: false,
        noSelfClose: true,
        extraSettings: {
          analytics: this.extraSettings.analytics,
          textSettings: {
            headingTextColor: '',
            textColorTheme: '',
            textStyleHeading: 'H4',
            textStyleParagraph: 'Paragraph',
            linksTextColor: 'var(--textAction)',
          },
        },
      };

      if (this.gameProcess === PUZZLES_GAME_PROCESS.complete) {
        return {
          ...baseProps,
          heading: 'Молодец!',
          buttonRepeat: {
            ...baseProps.buttonRepeat,
            text: 'Завершить игру'
          },
        };
      }

      if (this.gameProcess === PUZZLES_GAME_PROCESS.timeIsUp) {
        return {
          ...baseProps,
          heading: 'Внимание, время вышло.',
          buttonRepeat: {
            ...baseProps.buttonRepeat,
            text: 'Завершить игру'
          },
        };
      }

      return baseProps;
    },
    warningModalProps() {
      return {
        ...this.modalsSettings.warningModal,
        closeButton: false,
        noSelfClose: true,
        extraSettings: {
          analytics: this.extraSettings.analytics,
          textSettings: {
            headingTextColor: '',
            textColorTheme: '',
            textStyleHeading: 'H3',
            textStyleParagraph: 'Paragraph',
            linksTextColor: 'var(--textAction)',
          },
        },
      };
    },
    ratingModalProps() {
      return {
        heading: 'Таблица рейтинга',
        tableData: this.ratingData,
        extraSettings: {
          analytics: this.extraSettings.analytics,
          imgScale: this.extraSettings.imgScale,
          textSettings: {
            headingTextColor: '',
            textColorTheme: '',
            textStyleHeading: 'H3',
            textStyleParagraph: 'Paragraph',
            linksTextColor: 'var(--textAction)',
          },
        },
      };
    },
    accessToken() {
      // eslint-disable-next-line
      return this.siteData?.accessToken || null;
    },
    user() {
      // eslint-disable-next-line
      return this.siteData?.user || {};
    },
    userAccounts() {
      // eslint-disable-next-line
      return this.user.accounts || [];
    },
    userAccountSelected() {
      return this.userAccounts.find(account => account.selected) || {};
    },
    isUserAuthorizedAndSelectedAccount() {
      // eslint-disable-next-line
      return this.accessToken && Boolean(this.userAccountSelected?.id)
    },
    getLevelData() {
      return PUZZLES[this.activeLevel][this.activeLevelGameIndex];
    },
    unassembledLevels() {
      return this.allLevels.filter(level => !this.assembledLevels.includes(level));
    },
    firstUnassembledLevel() {
      return this.allLevels.find(level => !this.assembledLevels.includes(level));
    }
  },
  async serverPrefetch() {
    await this.loadIconsServer();
  },
  async beforeMount() {
    this.loadIconsClient();
    // await this.checkingUserData();
  },
  beforeDestroy() {
    this.siteData = null;
    this.isVisibleCompletedModal = false;
    this.isVisibleWarningModal = false;
  },
  methods: {
    async loadIconsServer() {
      const { iconKey } = this.buttonHome;

      if (iconKey) {
        this.$set(this.icons, iconKey, await this.loadIconServer(iconKey));
      }
    },
    loadIconsClient() {
      const { iconKey } = this.buttonHome;

      if (iconKey) {
        this.$set(this.icons, iconKey, this.loadIconClient(iconKey));
      }
    },
    getStorageData() {
      if (process.browser) {
        return storageHelper.getData(STORAGE_KEY);
      }

      return null;
    },
    async checkingUserData() {
      if (!this.isSigmaPreview) {
        const storageData = this.getStorageData();

        // eslint-disable-next-line
        if (storageData?.accessToken) {
          const { accessToken: token } = storageData;
          const { status } = await ApiService.getUserData(token);

          if (RESPONSE_STATUS_SUCCESS.includes(status)) {
            this.siteData = storageData;
            this.startGame();

            return;
          }
        }

        this.isVisibleWarningModal = true;
      }
    },
    async sendData() {
      if (this.isUserAuthorizedAndSelectedAccount) {
        await ApiService.postGameResult(
          this.accessToken,
          this.userAccountSelected.id,
          {
            gamecode: GAMES_CODES.Puzzles,
            points: this.points,
          }
        );
      }
    },

    async loadRatingData() {
      this.loadingRatingData = true;

      const response = await new Promise((resolve) => {
        setTimeout(() => {
          resolve({
            status: RESPONSE_STATUS_CODE.Ok,
            data: RATING_RAW,
          });
        }, 2000);
      });

      this.loadingRatingData = false;

      return response;
    },

    startGame() {
      this.isVisibleCompletedModal = false;

      this.isStartedGame = true;
      this.isStartTimer = true;
    },
    onCloseWarningModalHandler() {
      this.isVisibleWarningModal = false;
    },
    onCloseRatingModalHandler() {
      this.isVisibleRatingModal = false;
    },

    resetGameState() {
      this.assembledLevels = [];
      this.assembledPuzzlesActiveLevel = [];

      this.activeLevel = PUZZLES_LEVELS.easy;
      this.activeLevelGameIndex = 0;

      this.selectedElement = null;

      this.points = 0;

      this.isStartedGame = false;

      this.isStartTimer = false;
      this.isResetTimer = false;
      this.timeSeconds = Number(this.timer) || 300;
      this.passedSeconds = 0;
      this.remainingSeconds = Number(this.timer) || 300;
    },

    resetState() {
      this.isVisibleCompletedModal = false;
      this.isVisibleWarningModal = false;

      this.resetGameState();
    },

    onClickPuzzlesFieldElementButtonHandler(item) {
      if (!this.selectedElement) {
        return;
      }

      if (
        !this.assembledPuzzlesActiveLevel.includes(item.id) &&
        this.selectedElement.id === item.id
      ) {
        this.assembledPuzzlesActiveLevel.push(item.id);
        this.selectedElement = null;
        this.points += 10;

        if (this.assembledPuzzlesActiveLevel.length === this.getLevelData.items.length) {
          this.isVisibleCompletedModal = true;
        }
      }
    },
    onClickPuzzlesElementButtonHandler(item) {
      // eslint-disable-next-line
      if (this.selectedElement?.id !== item.id) {
        this.selectedElement = item;
      }
    },

    onClickNextPuzzleHandler() {
      if (this.gameProcess === PUZZLES_GAME_PROCESS.timeIsUp) {
        this.isVisibleCompletedModal = false;
        this.onClickCompleteHandler();

        return;
      }

      if (this.gameProcess === PUZZLES_GAME_PROCESS.complete) {
        this.isVisibleCompletedModal = false;
        this.onClickCompleteHandler();

        return;
      }

      if (this.unassembledLevels.length) {
        if (this.activeLevelGameIndex < PUZZLES[this.activeLevel].length - 1) {
          this.assembledPuzzlesActiveLevel = [];
          this.activeLevelGameIndex += 1;
        } else {
          if (!this.assembledLevels.includes(this.activeLevel)) {
            this.assembledLevels.push(this.activeLevel);
          }

          if (this.firstUnassembledLevel && this.firstUnassembledLevel !== this.activeLevel) {
            this.activeLevel = this.firstUnassembledLevel;
            this.assembledPuzzlesActiveLevel = [];
            this.activeLevelGameIndex = 0;
          }
        }
      }

      this.isVisibleCompletedModal = false;
    },

    async onClickCompleteHandler() {
      this.isResetTimer = true;
      this.isStartedGame = false;

      await this.sendData();

      const response = await this.loadRatingData();

      if (RESPONSE_STATUS_SUCCESS.includes(response.status)) {
        this.ratingData = this.processRatingData(response.data || [], GAME_AVATARS);

        this.isVisibleRatingModal = true;
      }

      this.resetGameState();
    },

    resetTimerHandler() {
      this.isResetTimer = false;
    },
    passedSecondsHandler(seconds) {
      this.passedSeconds = seconds;
    },
    remainingSecondsHandler(seconds) {
      this.remainingSeconds = seconds;
    },
    async timeIsUpHandler() {
      this.isResetTimer = true;
      this.isStartedGame = false;

      await this.sendData();

      this.gameProcess = PUZZLES_GAME_PROCESS.timeIsUp;
      this.isVisibleCompletedModal = true;
    },

    checkElementIsVisibleById(id) {
      return this.assembledPuzzlesActiveLevel.includes(id);
    },
    checkElementIsActiveById(id) {
      // eslint-disable-next-line
      return this.selectedElement?.id === id;
    },

    processRatingData(rawArray, avatars) {
      if (!rawArray.length) {
        return [];
      }

      return [...rawArray]
        .sort((a, b) => Number(b.points) - Number(a.points))
        .map(item => {
          const randomAvatar = avatars[Math.floor(Math.random() * avatars.length)];

          return {
            ...item,
            avatar: randomAvatar.img,
            points: Number(item.points).toLocaleString('ru-RU').replace(/\u00a0/g, ' ')
          };
        });
    },
  }
};
</script>
