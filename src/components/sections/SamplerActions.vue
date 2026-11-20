<script>
/**
 * Sampler Pad Actions.
 *
 * A sampler pad can run an action instead of playing a sample. The daemon stores the actions
 * (sampler-actions.json) and runs them, this page only edits `sampler_actions` through the websocket.
 * Starting programs (Launch / Open) is disabled by the daemon while network access is enabled.
 */
import { store } from "@/store";
import { websocket } from "@/util/sockets";
import GroupContainer from "@/components/containers/GroupContainer.vue";
import RadioSelection from "@/components/lists/RadioSelection.vue";
import CheckSelection from "@/components/lists/CheckSelection.vue";

const EVERY_BANK = "__every__";
const PADS = ["TopLeft", "TopRight", "BottomLeft", "BottomRight"];

function emptyDraft() {
  return {
    type: "None",
    command_line: "",
    working_dir: "",
    target: "",
    preset: "Preset1",
    profile: "",
    mic: false,
    json: "[]",
  };
}

export default {
  name: "SamplerActions",
  components: { GroupContainer, RadioSelection, CheckSelection },

  data() {
    return {
      bank: EVERY_BANK,
      pad: "TopLeft",
      draft: emptyDraft(),
      jsonError: "",
      saved: false,
    }
  },

  mounted() {
    this.loadDraft();
  },

  watch: {
    bank() { this.loadDraft(); },
    pad() { this.loadDraft(); },
  },

  methods: {
    getConfig() {
      return store.getActiveDevice().sampler_actions ?? { enabled: false, actions: [] };
    },

    isBlocked() {
      return store.getActiveDevice().launch_actions_blocked === true;
    },

    bankValue() {
      return this.bank === EVERY_BANK ? null : this.bank;
    },

    findExact() {
      let bank = this.bankValue();
      return this.getConfig().actions.find(a => a.bank === bank && a.button === this.pad);
    },

    /** Turns the stored action back into the editable draft */
    loadDraft() {
      let draft = emptyDraft();
      let entry = this.findExact();
      this.jsonError = "";
      this.saved = false;

      if (entry) {
        let type = Object.keys(entry.action)[0];
        let value = entry.action[type];
        draft.type = type;

        if (type === "Launch") {
          draft.command_line = value.command_line;
          draft.working_dir = value.working_dir ?? "";
        } else if (type === "Open") {
          draft.target = value.target;
        } else if (type === "SwitchEffectPreset") {
          draft.preset = value;
        } else if (type === "SwitchProfile") {
          draft.profile = value.name;
          draft.mic = value.mic;
        } else if (type === "Macro") {
          draft.json = JSON.stringify(value, null, 2);
        } else if (type === "Command") {
          // Shown as a macro with a single command
          draft.type = "Macro";
          draft.json = JSON.stringify([value], null, 2);
        }
      }
      this.draft = draft;
    },

    /** Selectors */
    getEnabledOptions() {
      return [{ id: "enabled", label: this.$t('message.samplerActions.enable'), selected: this.getConfig().enabled }];
    },

    onEnabledChanged(id, value) {
      websocket.send_command(store.getActiveSerial(), { "SetSamplerActionsEnabled": value });
    },

    getBankOptions() {
      return [
        { id: EVERY_BANK, label: this.$t('message.samplerActions.everyBank') },
        ...["A", "B", "C"].map(id => ({ id, label: this.$t('message.faderPaging.bank', { bank: id }) })),
      ];
    },

    getPadOptions() {
      return PADS.map(id => ({ id, label: this.$t('message.samplerActions.pads.' + id) }));
    },

    getTypeOptions() {
      let blocked = this.isBlocked();
      return [
        { id: "None", label: this.$t('message.samplerActions.types.None') },
        { id: "Launch", label: this.$t('message.samplerActions.types.Launch'), disabled: blocked },
        { id: "Open", label: this.$t('message.samplerActions.types.Open'), disabled: blocked },
        { id: "SwitchEffectPreset", label: this.$t('message.samplerActions.types.SwitchEffectPreset') },
        { id: "SwitchProfile", label: this.$t('message.samplerActions.types.SwitchProfile') },
        { id: "Macro", label: this.$t('message.samplerActions.types.Macro') },
      ];
    },

    onTypeChanged(id) {
      this.draft.type = id;
      this.saved = false;
    },

    getPresetOptions() {
      return [1, 2, 3, 4, 5, 6].map(n => ({ id: "Preset" + n, label: this.$t('message.samplerActions.preset', { n }) }));
    },

    getProfileOptions() {
      let names = this.draft.mic ? store.getMicProfileFiles() : store.getProfileFiles();
      return [...names].sort(Intl.Collator().compare).map(name => ({ id: name, label: name }));
    },

    getProfileKindOptions() {
      return [{ id: "mic", label: this.$t('message.samplerActions.micProfile'), selected: this.draft.mic }];
    },

    onProfileKindChanged(id, value) {
      this.draft.mic = value;
      this.draft.profile = "";
    },

    /** Summary list */
    describe(entry) {
      let type = Object.keys(entry.action)[0];
      let value = entry.action[type];
      let text = this.$t('message.samplerActions.types.' + type);
      if (type === "Launch") text += ": " + value.command_line;
      else if (type === "Open") text += ": " + value.target;
      else if (type === "SwitchEffectPreset") text += ": " + value;
      else if (type === "SwitchProfile") text += ": " + value.name;
      else if (type === "Macro") text += " (" + value.length + ")";
      return text;
    },

    bankLabel(entry) {
      return entry.bank === null ? this.$t('message.samplerActions.everyBank') : this.$t('message.faderPaging.bank', { bank: entry.bank });
    },

    select(entry) {
      this.bank = entry.bank === null ? EVERY_BANK : entry.bank;
      this.pad = entry.button;
    },

    /** Saving */
    buildAction() {
      let d = this.draft;
      switch (d.type) {
        case "Launch":
          return { "Launch": { command_line: d.command_line, working_dir: d.working_dir.trim() === "" ? null : d.working_dir } };
        case "Open":
          return { "Open": { target: d.target } };
        case "SwitchEffectPreset":
          return { "SwitchEffectPreset": d.preset };
        case "SwitchProfile":
          return { "SwitchProfile": { name: d.profile, mic: d.mic } };
        case "Macro": {
          let commands = JSON.parse(d.json);
          if (!Array.isArray(commands)) throw new Error("Expected a list of commands");
          return { "Macro": commands };
        }
      }
      return null;
    },

    save() {
      this.jsonError = "";
      if (this.draft.type === "None") {
        this.clear();
        return;
      }

      let action;
      try {
        action = this.buildAction();
      } catch (e) {
        this.jsonError = this.$t('message.samplerActions.invalidJson') + " " + e.message;
        return;
      }

      websocket.send_command(store.getActiveSerial(), { "SetSamplerAction": [this.bankValue(), this.pad, action] });
      this.saved = true;
    },

    clear() {
      websocket.send_command(store.getActiveSerial(), { "ClearSamplerAction": [this.bankValue(), this.pad] });
      this.draft = emptyDraft();
      this.saved = false;
    },

    test() {
      websocket.send_command(store.getActiveSerial(), { "TestSamplerAction": [this.bankValue(), this.pad] });
    },
  }
}
</script>

<template>
  <div style="display: flex">
    <div style="display: flex; flex-wrap: wrap; padding: 15px 20px; gap: 20px; width: 100%">
      <GroupContainer :title="$t('message.samplerActions.title')">
        <CheckSelection :title="$t('message.samplerActions.general')" :options="getEnabledOptions()"
                        @selection-changed="onEnabledChanged"/>
        <div v-if="isBlocked()" class="notice">{{ $t('message.samplerActions.blocked') }}</div>
        <RadioSelection :title="$t('message.samplerActions.bankTitle')" group="sampler_actions_bank"
                        :options="getBankOptions()" :selected="bank" @selection-changed="id => bank = id"/>
        <RadioSelection :title="$t('message.samplerActions.padTitle')" group="sampler_actions_pad"
                        :options="getPadOptions()" :selected="pad" @selection-changed="id => pad = id"/>
      </GroupContainer>

      <GroupContainer :title="$t('message.samplerActions.actionTitle')">
        <RadioSelection :title="$t('message.samplerActions.typeTitle')" group="sampler_actions_type"
                        :options="getTypeOptions()" :selected="draft.type" @selection-changed="onTypeChanged"/>

        <div class="form">
          <template v-if="draft.type === 'Launch'">
            <label>{{ $t('message.samplerActions.commandLine') }}</label>
            <input type="text" v-model="draft.command_line" placeholder="C:\Program Files\App\app.exe --flag"/>
            <label>{{ $t('message.samplerActions.workingDir') }}</label>
            <input type="text" v-model="draft.working_dir"/>
          </template>

          <template v-if="draft.type === 'Open'">
            <label>{{ $t('message.samplerActions.target') }}</label>
            <input type="text" v-model="draft.target" placeholder="https://..."/>
          </template>

          <template v-if="draft.type === 'Macro'">
            <label>{{ $t('message.samplerActions.commandsJson') }}</label>
            <textarea v-model="draft.json" rows="8" spellcheck="false"></textarea>
          </template>

          <div v-if="jsonError" class="error">{{ jsonError }}</div>

          <div class="buttons">
            <button @click="save">{{ $t('message.samplerActions.save') }}</button>
            <button @click="test" :disabled="!findExact()">{{ $t('message.samplerActions.test') }}</button>
            <button @click="clear" :disabled="!findExact()">{{ $t('message.samplerActions.clear') }}</button>
            <span v-if="saved" class="saved">{{ $t('message.samplerActions.saved') }}</span>
          </div>
        </div>

        <RadioSelection v-if="draft.type === 'SwitchEffectPreset'" :title="$t('message.samplerActions.presetTitle')"
                        group="sampler_actions_preset" :options="getPresetOptions()" :selected="draft.preset"
                        @selection-changed="id => draft.preset = id"/>

        <template v-if="draft.type === 'SwitchProfile'">
          <CheckSelection :title="$t('message.samplerActions.profileKind')" :options="getProfileKindOptions()"
                          @selection-changed="onProfileKindChanged"/>
          <RadioSelection :title="$t('message.samplerActions.profileTitle')" group="sampler_actions_profile"
                          :options="getProfileOptions()" :selected="draft.profile"
                          @selection-changed="id => draft.profile = id"/>
        </template>
      </GroupContainer>

      <GroupContainer :title="$t('message.samplerActions.overviewTitle')">
        <div class="overview">
          <div v-if="getConfig().actions.length === 0" class="empty">{{ $t('message.samplerActions.none') }}</div>
          <div v-for="(entry, index) in getConfig().actions" :key="index" class="row" @click="select(entry)">
            <span class="where">{{ bankLabel(entry) }} / {{ $t('message.samplerActions.pads.' + entry.button) }}</span>
            <span class="what">{{ describe(entry) }}</span>
          </div>
        </div>
      </GroupContainer>
    </div>
  </div>
</template>

<style scoped>
.form, .overview, .notice {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px;
  color: #fff;
  min-width: 260px;
}

.notice {
  background-color: #5a3b2a;
  max-width: 260px;
}

label {
  color: #fff;
}

input[type=text], textarea {
  font-family: LeagueMonoCondensed, sans-serif;
  background-color: #3b413f;
  color: #59b1b6;
  box-sizing: border-box;
  padding: 10px;
  border: none;
  outline: none;
  width: 100%;
}

textarea {
  resize: vertical;
}

.buttons {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.buttons button {
  background-color: #3b413f;
  color: #fff;
  border: 0;
  padding: 8px 14px;
  cursor: pointer;
}

.buttons button:hover:not(:disabled) {
  background-color: #49514e;
}

.buttons button:disabled {
  opacity: 0.4;
  cursor: default;
}

.saved {
  color: #59b1b6;
}

.error {
  color: #ff8a80;
}

.overview .row {
  display: flex;
  flex-direction: column;
  padding: 6px 8px;
  background-color: #3b413f;
  cursor: pointer;
}

.overview .row:hover {
  background-color: #49514e;
}

.overview .where {
  color: #59b1b6;
}

.overview .empty {
  opacity: 0.7;
}
</style>
