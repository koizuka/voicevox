<template>
  <menu-bar />

  <q-layout reveal elevated>
    <header-bar />

    <q-page-container>
      <q-page class="main-row-panes bg-background">
        <div v-if="engineState === 'STARTING'" class="waiting-engine">
          <div>
            <q-spinner color="primary" size="2.5rem" />
            <div>エンジン起動中・・・</div>
          </div>
        </div>
        <q-splitter
          horizontal
          reverse
          unit="px"
          :limits="[audioDetailPaneMinHeight, audioDetailPaneMaxHeight]"
          separator-class="bg-primary"
          :separator-style="{ height: shouldShowPanes ? '3px' : 0 }"
          class="full-width"
          before-class="overflow-hidden"
          :disable="!shouldShowPanes"
          v-model="audioDetailPaneHeight"
        >
          <template #before>
            <q-splitter
              :limits="[MIN_PORTRAIT_PANE_WIDTH, MAX_PORTRAIT_PANE_WIDTH]"
              separator-class="bg-primary"
              :separator-style="{ width: shouldShowPanes ? '3px' : 0 }"
              before-class="overflow-hidden"
              :disable="!shouldShowPanes"
              v-model="portraitPaneWidth"
            >
              <template #before>
                <character-portrait />
              </template>
              <template #after>
                <q-splitter
                  reverse
                  unit="px"
                  :limits="[audioInfoPaneMinWidth, audioInfoPaneMaxWidth]"
                  separator-class="bg-primary"
                  :separator-style="{ width: shouldShowPanes ? '3px' : 0 }"
                  class="full-width overflow-hidden"
                  :disable="!shouldShowPanes"
                  v-model="audioInfoPaneWidth"
                >
                  <template #before>
                    <div
                      class="audio-cell-pane"
                      :class="{ 'is-dragging': dragEventCounter > 0 }"
                      @dragenter="dragEventCounter++"
                      @dragleave="dragEventCounter--"
                      @dragover.prevent
                      @drop.prevent="
                        dragEventCounter = 0;
                        loadDraggedFile($event);
                      "
                    >
                      <div class="audio-cells">
                        <audio-cell
                          v-for="audioKey in audioKeys"
                          :key="audioKey"
                          :audioKey="audioKey"
                          :ref="addAudioCellRef"
                          @focusCell="focusCell"
                        />
                      </div>
                      <div class="add-button-wrapper">
                        <q-btn
                          fab
                          icon="add"
                          color="primary-light"
                          text-color="display-dark"
                          :disable="uiLocked"
                          @click="addAudioItem"
                        ></q-btn>
                      </div>
                    </div>
                  </template>
                  <template #after>
                    <audio-info
                      v-if="activeAudioKey != undefined"
                      :activeAudioKey="activeAudioKey"
                    />
                  </template>
                </q-splitter>
              </template>
            </q-splitter>
          </template>
          <template #after>
            <audio-detail
              v-if="activeAudioKey != undefined"
              :activeAudioKey="activeAudioKey"
            />
          </template>
        </q-splitter>

        <q-resize-observer
          ref="resizeObserverRef"
          @resize="({ height }) => changeAudioDetailPaneMaxHeight(height)"
        />
      </q-page>
    </q-page-container>
  </q-layout>
  <help-dialog v-model="isHelpDialogOpenComputed" />
  <setting-dialog v-model="isSettingDialogOpenComputed" />
  <hotkey-setting-dialog v-model="isHotkeySettingDialogOpenComputed" />
  <default-style-select-dialog
    v-if="characterInfos"
    :characterInfos="characterInfos"
    v-model="isDefaultStyleSelectDialogOpenComputed"
  />
</template>

<script lang="ts">
import {
  computed,
  defineComponent,
  onBeforeUpdate,
  onMounted,
  ref,
  watch,
} from "vue";
import { useStore } from "@/store";
import HeaderBar from "@/components/HeaderBar.vue";
import AudioCell from "@/components/AudioCell.vue";
import AudioDetail from "@/components/AudioDetail.vue";
import AudioInfo from "@/components/AudioInfo.vue";
import MenuBar from "@/components/MenuBar.vue";
import HelpDialog from "@/components/HelpDialog.vue";
import SettingDialog from "@/components/SettingDialog.vue";
import HotkeySettingDialog from "@/components/HotkeySettingDialog.vue";
import CharacterPortrait from "@/components/CharacterPortrait.vue";
import DefaultStyleSelectDialog from "@/components/DefaultStyleSelectDialog.vue";
import { AudioItem } from "@/store/type";
import { QResizeObserver } from "quasar";
import path from "path";
import { HotkeyAction, HotkeyReturnType } from "@/type/preload";
import { parseCombo, setHotkeyFunctions } from "@/store/setting";

export default defineComponent({
  name: "Home",

  components: {
    MenuBar,
    HeaderBar,
    AudioCell,
    AudioDetail,
    AudioInfo,
    HelpDialog,
    SettingDialog,
    HotkeySettingDialog,
    CharacterPortrait,
    DefaultStyleSelectDialog,
  },

  setup() {
    const store = useStore();

    const audioItems = computed(() => store.state.audioItems);
    const audioKeys = computed(() => store.state.audioKeys);
    const uiLocked = computed(() => store.getters.UI_LOCKED);

    // hotkeys handled by Mousetrap
    const hotkeyMap = new Map<HotkeyAction, () => HotkeyReturnType>([
      [
        "テキスト欄にフォーカスを戻す",
        () => {
          if (activeAudioKey.value !== undefined) {
            focusCell({ audioKey: activeAudioKey.value });
          }
          return false; // this is the same with event.preventDefault()
        },
      ],
    ]);

    setHotkeyFunctions(hotkeyMap);

    const removeAudioItem = async () => {
      if (activeAudioKey.value == undefined) throw new Error();
      audioCellRefs[activeAudioKey.value].removeCell();
    };

    // convert the hotkey array to Map to get value with keys easier
    // this only happens here where we deal with native methods
    const hotkeySettingsMap = computed(
      () =>
        new Map(
          store.state.hotkeySettings.map((obj) => [obj.action, obj.combination])
        )
    );

    // hotkeys handled by native, for they are made to be working while focusing input elements
    const hotkeyActionsNative = [
      (event: KeyboardEvent) => {
        if (
          !event.isComposing &&
          !uiLocked.value &&
          parseCombo(event) == hotkeySettingsMap.value.get("テキスト欄を追加")
        ) {
          addAudioItem();
          event.preventDefault();
        }
      },
      (event: KeyboardEvent) => {
        if (
          !event.isComposing &&
          !uiLocked.value &&
          parseCombo(event) == hotkeySettingsMap.value.get("テキスト欄を削除")
        ) {
          removeAudioItem();
          event.preventDefault();
        }
      },
      (event: KeyboardEvent) => {
        if (
          !event.isComposing &&
          !uiLocked.value &&
          parseCombo(event) ==
            hotkeySettingsMap.value.get("テキスト欄からフォーカスを外す")
        ) {
          if (document.activeElement instanceof HTMLInputElement) {
            document.activeElement.blur();
          }
          event.preventDefault();
        }
      },
    ];

    // view
    const DEFAULT_PORTRAIT_PANE_WIDTH = 25; // %
    const MIN_PORTRAIT_PANE_WIDTH = 0;
    const MAX_PORTRAIT_PANE_WIDTH = 40;
    const MIN_AUDIO_INFO_PANE_WIDTH = 160; // px
    const MAX_AUDIO_INFO_PANE_WIDTH = 250;
    const MIN_AUDIO_DETAIL_PANE_HEIGHT = 185; // px
    const MAX_AUDIO_DETAIL_PANE_HEIGHT = 500;

    const portraitPaneWidth = ref(0);
    const audioInfoPaneWidth = ref(0);
    const audioInfoPaneMinWidth = ref(0);
    const audioInfoPaneMaxWidth = ref(0);
    const audioDetailPaneHeight = ref(0);
    const audioDetailPaneMinHeight = ref(0);
    const audioDetailPaneMaxHeight = ref(0);

    const changeAudioDetailPaneMaxHeight = (height: number) => {
      if (!activeAudioKey.value) return;

      const maxHeight = height - 200;
      if (maxHeight > MAX_AUDIO_DETAIL_PANE_HEIGHT) {
        // 最大値以上なら最大値に設定
        audioDetailPaneMaxHeight.value = MAX_AUDIO_DETAIL_PANE_HEIGHT;
      } else if (height < 200 + MIN_AUDIO_DETAIL_PANE_HEIGHT) {
        // 最低値以下になってしまう場合は無制限に
        audioDetailPaneMaxHeight.value = Infinity;
      } else {
        audioDetailPaneMaxHeight.value = maxHeight;
      }
    };

    // component
    let audioCellRefs: Record<string, typeof AudioCell> = {};
    const addAudioCellRef = (audioCellRef: typeof AudioCell) => {
      if (audioCellRef) {
        audioCellRefs[audioCellRef.audioKey] = audioCellRef;
      }
    };
    onBeforeUpdate(() => {
      audioCellRefs = {};
    });

    const resizeObserverRef = ref<QResizeObserver>();

    // セルを追加
    const activeAudioKey = computed<string | undefined>(
      () => store.getters.ACTIVE_AUDIO_KEY
    );
    const addAudioItem = async () => {
      const prevAudioKey = activeAudioKey.value;
      let styleId: number | undefined = undefined;
      if (prevAudioKey !== undefined) {
        styleId = store.state.audioItems[prevAudioKey].styleId;
      }
      let audioItem: AudioItem;
      let baseAudioItem: AudioItem | undefined = undefined;
      if (store.state.inheritAudioInfo) {
        baseAudioItem = prevAudioKey
          ? store.state.audioItems[prevAudioKey]
          : undefined;
      }
      //パラメータ引き継ぎがONの場合は話速等のパラメータを引き継いでテキスト欄を作成する
      //パラメータ引き継ぎがOFFの場合、baseAudioItemがundefinedになっているのでパラメータ引き継ぎは行われない
      audioItem = await store.dispatch("GENERATE_AUDIO_ITEM", {
        styleId,
        baseAudioItem,
      });

      const newAudioKey = await store.dispatch("COMMAND_REGISTER_AUDIO_ITEM", {
        audioItem,
        prevAudioKey: activeAudioKey.value,
      });
      audioCellRefs[newAudioKey].focusTextField();
    };

    // Pane
    const shouldShowPanes = computed<boolean>(
      () => store.getters.SHOULD_SHOW_PANES
    );
    watch(shouldShowPanes, (val, old) => {
      if (val === old) return;

      if (val) {
        portraitPaneWidth.value = DEFAULT_PORTRAIT_PANE_WIDTH;
        audioInfoPaneWidth.value = MIN_AUDIO_INFO_PANE_WIDTH;
        audioInfoPaneMinWidth.value = MIN_AUDIO_INFO_PANE_WIDTH;
        audioInfoPaneMaxWidth.value = MAX_AUDIO_INFO_PANE_WIDTH;
        audioDetailPaneHeight.value = MIN_AUDIO_DETAIL_PANE_HEIGHT;
        audioDetailPaneMinHeight.value = MIN_AUDIO_DETAIL_PANE_HEIGHT;
        changeAudioDetailPaneMaxHeight(
          resizeObserverRef.value?.$el.parentElement.clientHeight
        );
      } else {
        portraitPaneWidth.value = 0;
        audioInfoPaneWidth.value = 0;
        audioInfoPaneMinWidth.value = 0;
        audioInfoPaneMaxWidth.value = 0;
        audioDetailPaneHeight.value = 0;
        audioDetailPaneMinHeight.value = 0;
        audioDetailPaneMaxHeight.value = 0;
      }
    });

    // セルをフォーカス
    const focusCell = ({ audioKey }: { audioKey: string }) => {
      audioCellRefs[audioKey].focusTextField();
    };

    const disableDefaultUndoRedo = (event: KeyboardEvent) => {
      // ctrl+z, ctrl+shift+z, ctrl+y
      if (
        event.ctrlKey &&
        (event.key == "z" || (!event.shiftKey && event.key == "y"))
      ) {
        event.preventDefault();
      }
    };

    // プロジェクトを初期化
    onMounted(async () => {
      await Promise.all([
        store.dispatch("LOAD_CHARACTER"),
        store.dispatch("LOAD_DEFAULT_STYLE_IDS"),
      ]);
      if (await store.dispatch("IS_UNSET_DEFAULT_STYLE_IDS")) {
        isDefaultStyleSelectDialogOpenComputed.value = true;
      }
      const audioItem: AudioItem = await store.dispatch(
        "GENERATE_AUDIO_ITEM",
        {}
      );
      const newAudioKey = await store.dispatch("REGISTER_AUDIO_ITEM", {
        audioItem,
      });
      focusCell({ audioKey: newAudioKey });

      document.addEventListener("keydown", disableDefaultUndoRedo);

      hotkeyActionsNative.forEach((item) => {
        document.addEventListener("keyup", item);
      });
    });

    // エンジン待機
    const engineState = computed(() => store.state.engineState);

    // ライセンス表示
    const isHelpDialogOpenComputed = computed({
      get: () => store.state.isHelpDialogOpen,
      set: (val) =>
        store.dispatch("IS_HELP_DIALOG_OPEN", { isHelpDialogOpen: val }),
    });

    // 設定
    const isSettingDialogOpenComputed = computed({
      get: () => store.state.isSettingDialogOpen,
      set: (val) =>
        store.dispatch("IS_SETTING_DIALOG_OPEN", { isSettingDialogOpen: val }),
    });

    // ショートカットキー設定
    const isHotkeySettingDialogOpenComputed = computed({
      get: () => store.state.isHotkeySettingDialogOpen,
      set: (val) =>
        store.dispatch("IS_HOTKEY_SETTING_DIALOG_OPEN", {
          isHotkeySettingDialogOpen: val,
        }),
    });

    // デフォルトスタイル選択
    const characterInfos = computed(() => store.state.characterInfos);
    const isDefaultStyleSelectDialogOpenComputed = computed({
      get: () => store.state.isDefaultStyleSelectDialogOpen,
      set: (val) =>
        store.dispatch("IS_DEFAULT_STYLE_SELECT_DIALOG_OPEN", {
          isDefaultStyleSelectDialogOpen: val,
        }),
    });

    // ドラッグ＆ドロップ
    const dragEventCounter = ref(0);
    const loadDraggedFile = (event?: { dataTransfer: DataTransfer }) => {
      if (!event || event.dataTransfer.files.length === 0) return;
      const file = event.dataTransfer.files[0];
      switch (path.extname(file.name)) {
        case ".txt":
          store.dispatch("COMMAND_IMPORT_FROM_FILE", { filePath: file.path });
          break;
        case ".vvproj":
          store.dispatch("LOAD_PROJECT_FILE", { filePath: file.path });
          break;
        default:
          store.dispatch("SHOW_WARNING_DIALOG", {
            title: "対応していないファイルです",
            message:
              "テキストファイル (.txt) とVOICEVOXプロジェクトファイル (.vvproj) に対応しています。",
          });
      }
    };

    return {
      audioItems,
      audioKeys,
      uiLocked,
      addAudioCellRef,
      activeAudioKey,
      addAudioItem,
      shouldShowPanes,
      focusCell,
      changeAudioDetailPaneMaxHeight,
      resizeObserverRef,
      MIN_PORTRAIT_PANE_WIDTH,
      MAX_PORTRAIT_PANE_WIDTH,
      portraitPaneWidth,
      audioInfoPaneWidth,
      audioInfoPaneMinWidth,
      audioInfoPaneMaxWidth,
      audioDetailPaneHeight,
      audioDetailPaneMinHeight,
      audioDetailPaneMaxHeight,
      engineState,
      isHelpDialogOpenComputed,
      isSettingDialogOpenComputed,
      isHotkeySettingDialogOpenComputed,
      characterInfos,
      isDefaultStyleSelectDialogOpenComputed,
      dragEventCounter,
      loadDraggedFile,
    };
  },
});
</script>

<style lang="scss">
@use '@/styles' as global;
body {
  user-select: none;
  border-left: solid #{global.$window-border-width} #{global.$primary};
  border-right: solid #{global.$window-border-width} #{global.$primary};
  border-bottom: solid #{global.$window-border-width} #{global.$primary};
}

.relative-absolute-wrapper {
  position: relative;
  > div {
    position: absolute;
    inset: 0;
  }
}
</style>

<style lang="scss">
@use '@/styles' as global;

.q-header {
  height: global.$header-height;
}

.waiting-engine {
  background-color: var(--color-background);
  position: absolute;
  inset: 0;
  z-index: 10;
  display: flex;
  text-align: center;
  align-items: center;
  justify-content: center;

  > div {
    color: var(--color-display-dark);
    background: var(--color-background-light);
    border-radius: 6px;
    padding: 14px;
  }
}

.main-row-panes {
  flex-grow: 1;
  flex-shrink: 1;
  flex-basis: 0;

  display: flex;

  .q-splitter--horizontal {
    height: calc(
      100vh - #{global.$menubar-height + global.$header-height +
        global.$window-border-width}
    );
  }
}

.audio-cell-pane {
  flex-grow: 1;
  flex-shrink: 1;
  flex-basis: 0;

  position: relative;
  height: 100%;

  &.is-dragging {
    background-color: var(--color-background);
  }

  .audio-cells {
    overflow-x: hidden;
    overflow-y: scroll;

    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;

    padding-bottom: 70px;
  }
  .add-button-wrapper {
    position: absolute;
    right: 0px;
    bottom: 0px;

    margin-right: 26px;
    margin-bottom: 10px;
  }
}
</style>
