<script>
/**
 * Fader Paging configuration.
 *
 * The Sampler Banks (A / B / C) act as 'pages' for the four faders. The daemon stores the pages
 * (fader-pages.json) and applies them whenever the active bank changes, so nothing here runs
 * on the device, we simply edit `fader_paging` in the daemon status through the websocket.
 *
 * Every property of a fader entry is optional, 'null' means the profile is left untouched.
 */
import { store } from "@/store";
import { websocket } from "@/util/sockets";
import { MuteButtonNamesForFader, ScribbleNames, getLightingInactiveOptions } from "@/util/mixerMapping";
import { firmwareSupportsMix2 } from "@/util/util";
import GroupContainer from "@/components/containers/GroupContainer.vue";
import RadioSelection from "@/components/lists/RadioSelection.vue";
import CheckSelection from "@/components/lists/CheckSelection.vue";
import ColourPicker from "@/components/sections/lighting/elements/ColourPicker.vue";

const UNCHANGED = "__unchanged__";

export default {
  name: "FaderPaging",
  components: { GroupContainer, RadioSelection, CheckSelection, ColourPicker },

  data() {
    return {
      activeBank: "A",
      activeFader: "A",
      textValue: "",
    }
  },

  mounted() {
    this.activeBank = store.getActiveDevice().sampler?.active_bank ?? "A";
    this.textValue = this.getScribble()?.text ?? "";
  },

  watch: {
    activeFader() { this.textValue = this.getScribble()?.text ?? ""; },
    activeBank() { this.textValue = this.getScribble()?.text ?? ""; },
  },

  methods: {
    getLightingInactiveOptions,

    /** Config access **/
    getConfig() {
      return store.getActiveDevice().fader_paging ?? { enabled: false, pages: [] };
    },

    getEntry() {
      let page = this.getConfig().pages.find(p => p.bank === this.activeBank);
      let entry = page?.faders.find(f => f.fader === this.activeFader);
      // Always return a copy, we never want to edit the status directly.
      return JSON.parse(JSON.stringify(entry ?? this.emptyEntry()));
    },

    emptyEntry() {
      return {
        fader: this.activeFader,
        channel: null,
        mute_function: null,
        display_style: null,
        fader_colours: null,
        mute_colours: null,
        mute_off_style: null,
        scribble: null,
      };
    },

    save(entry) {
      websocket.send_command(store.getActiveSerial(), { "SetFaderPageEntry": [this.activeBank, entry] });
    },

    /** Current device state, used to seed a section when it's switched on **/
    device() {
      return store.getActiveDevice();
    },

    /** Master switch / page tools **/
    getEnabledOptions() {
      return [{ id: "enabled", label: this.$t('message.faderPaging.enable'), selected: this.getConfig().enabled }];
    },

    onEnabledChanged(id, value) {
      websocket.send_command(store.getActiveSerial(), { "SetFaderPagingEnabled": value });
    },

    captureBank() {
      websocket.send_command(store.getActiveSerial(), { "CaptureFaderPage": this.activeBank });
    },

    applyBank() {
      websocket.send_command(store.getActiveSerial(), { "ApplyFaderPage": this.activeBank });
    },

    clearEntry() {
      websocket.send_command(store.getActiveSerial(), { "ClearFaderPageEntry": [this.activeBank, this.activeFader] });
    },

    /** Selectors **/
    getBankOptions() {
      return ["A", "B", "C"].map(id => ({ id, label: this.$t('message.faderPaging.bank', { bank: id }) }));
    },

    getFaderOptions() {
      return ["A", "B", "C", "D"].map(id => ({ id, label: this.$t('message.faders.' + id) }));
    },

    /** Channel **/
    getChannelOptions() {
      let options = [{ id: UNCHANGED, label: this.$t('message.faderPaging.unchanged') }];
      for (let id of ["Mic", "Chat", "Music", "Game", "Console", "LineIn", "System", "Sample", "Headphones", "LineOut"]) {
        options.push({ id, label: this.$t('message.channels.' + id) });
      }
      return options;
    },

    getChannel() {
      return this.getEntry().channel ?? UNCHANGED;
    },

    onChannelChanged(id) {
      let entry = this.getEntry();
      entry.channel = id === UNCHANGED ? null : id;
      this.save(entry);
    },

    /** Mute behaviour **/
    getMuteFunctionOptions() {
      let options = [
        { id: UNCHANGED, label: this.$t('message.faderPaging.unchanged') },
        { id: "All", label: this.$t('message.configuration.mute_behaviour.all') },
        { id: "ToStream", label: this.$t('message.configuration.mute_behaviour.stream') },
      ];
      if (firmwareSupportsMix2()) {
        options.push({ id: "ToStream2", label: this.$t('message.configuration.mute_behaviour.stream2') });
        options.push({ id: "ToStreams", label: this.$t('message.configuration.mute_behaviour.streams') });
      }
      options.push({ id: "ToVoiceChat", label: this.$t('message.configuration.mute_behaviour.chatMic') });
      options.push({ id: "ToPhones", label: this.$t('message.configuration.mute_behaviour.headphones') });
      options.push({ id: "ToLineOut", label: this.$t('message.configuration.mute_behaviour.lineOut') });
      return options;
    },

    getMuteFunction() {
      return this.getEntry().mute_function ?? UNCHANGED;
    },

    onMuteFunctionChanged(id) {
      let entry = this.getEntry();
      entry.mute_function = id === UNCHANGED ? null : id;
      this.save(entry);
    },

    /** Fader lighting **/
    getFaderLightingOptions() {
      let style = this.getEntry().display_style;
      return [
        { id: "override", label: this.$t('message.faderPaging.override'), selected: style !== null },
        { id: "gradient", label: this.$t('message.lighting.mixer.fader.styles.gradient'), selected: style !== null && style.includes("Gradient"), disabled: style === null },
        { id: "meter", label: this.$t('message.lighting.mixer.fader.styles.meter'), selected: style !== null && style.includes("Meter"), disabled: style === null },
      ];
    },

    onFaderLightingChanged(id, value) {
      let entry = this.getEntry();
      if (id === "override") {
        if (value) {
          // Seed from what the device is currently showing
          let light = this.device().lighting.faders[this.activeFader];
          entry.display_style = light.style;
          entry.fader_colours = [light.colours.colour_one, light.colours.colour_two];
        } else {
          entry.display_style = null;
          entry.fader_colours = null;
        }
      } else {
        let gradient = entry.display_style.includes("Gradient");
        let meter = entry.display_style.includes("Meter");
        if (id === "gradient") gradient = value;
        if (id === "meter") meter = value;
        entry.display_style = gradient && meter ? "GradientMeter" : gradient ? "Gradient" : meter ? "Meter" : "TwoColour";
      }
      this.save(entry);
    },

    hasFaderColours() {
      return this.getEntry().fader_colours !== null;
    },

    getFaderColour(index) {
      let colours = this.getEntry().fader_colours;
      return "#" + (colours ? colours[index] : "000000");
    },

    onFaderColourChanged(index, value) {
      let entry = this.getEntry();
      if (entry.fader_colours === null) return;
      entry.fader_colours[index] = value.substr(1, 6);
      this.save(entry);
    },

    /** Mute button lighting **/
    getMuteLightingOptions() {
      return [{ id: "override", label: this.$t('message.faderPaging.override'), selected: this.getEntry().mute_colours !== null }];
    },

    onMuteLightingChanged(id, value) {
      let entry = this.getEntry();
      if (value) {
        let light = this.device().lighting.buttons[MuteButtonNamesForFader[this.activeFader]];
        entry.mute_colours = [light.colours.colour_one, light.colours.colour_two];
        entry.mute_off_style = light.off_style;
      } else {
        entry.mute_colours = null;
        entry.mute_off_style = null;
      }
      this.save(entry);
    },

    hasMuteColours() {
      return this.getEntry().mute_colours !== null;
    },

    getMuteColour(index) {
      let colours = this.getEntry().mute_colours;
      return "#" + (colours ? colours[index] : "000000");
    },

    onMuteColourChanged(index, value) {
      let entry = this.getEntry();
      if (entry.mute_colours === null) return;
      entry.mute_colours[index] = value.substr(1, 6);
      this.save(entry);
    },

    getMuteOffStyle() {
      return this.getEntry().mute_off_style ?? "Dimmed";
    },

    onMuteOffStyleChanged(id) {
      let entry = this.getEntry();
      if (entry.mute_colours === null) return;
      entry.mute_off_style = id;
      this.save(entry);
    },

    /** Scribble **/
    getScribble() {
      return this.getEntry().scribble;
    },

    hasScribble() {
      return this.getScribble() !== null;
    },

    getScribbleOptions() {
      let scribble = this.getScribble();
      return [
        { id: "override", label: this.$t('message.faderPaging.override'), selected: scribble !== null },
        { id: "invert", label: this.$t('message.lighting.mixer.screen.options.invertDisplay'), selected: scribble?.inverted ?? false, disabled: scribble === null },
        { id: "number", label: this.$t('message.lighting.mixer.screen.options.showNumber'), selected: !!scribble?.number, disabled: scribble === null },
      ];
    },

    onScribbleOptionsChanged(id, value) {
      let entry = this.getEntry();
      if (id === "override") {
        if (value) {
          // Seed from the device
          let current = this.device().fader_status[this.activeFader].scribble;
          entry.scribble = {
            icon: current?.file_name ?? null,
            text: current?.bottom_text ?? null,
            number: current?.left_text ?? null,
            inverted: current?.inverted ?? false,
            colour: this.device().lighting.simple[ScribbleNames[this.activeFader]].colour_one,
          };
        } else {
          entry.scribble = null;
        }
      } else if (id === "invert") {
        entry.scribble.inverted = value;
      } else if (id === "number") {
        let index = ["A", "B", "C", "D"].indexOf(this.activeFader) + 1;
        entry.scribble.number = value ? index.toString() : "";
      }
      this.save(entry);
      this.textValue = entry.scribble?.text ?? "";
    },

    getScribbleColour() {
      return "#" + (this.getScribble()?.colour ?? "000000");
    },

    onScribbleColourChanged(value) {
      let entry = this.getEntry();
      if (entry.scribble === null) return;
      entry.scribble.colour = value.substr(1, 6);
      this.save(entry);
    },

    getIconOptions() {
      let options = [{ id: null, label: this.$t('message.lighting.mixer.screen.iconNone') }];
      [...store.getIconFiles()].sort(Intl.Collator().compare).forEach(item => options.push({ id: item, label: item }));
      return options;
    },

    getScribbleIcon() {
      return this.getScribble()?.icon ?? null;
    },

    onScribbleIconChanged(value) {
      let entry = this.getEntry();
      if (entry.scribble === null) return;
      entry.scribble.icon = value;
      this.save(entry);
    },

    onScribbleTextApplied() {
      let entry = this.getEntry();
      if (entry.scribble === null) return;
      // Empty means 'show the channel name' (handled by the daemon)
      entry.scribble.text = this.textValue === "" ? null : this.textValue;
      this.save(entry);
    },
  }
}
</script>

<template>
  <div style="display: flex">
    <div style="display: flex; flex-wrap: wrap; padding: 15px 20px; gap: 20px; width: 100%">
      <GroupContainer :title="$t('message.faderPaging.title')">
        <CheckSelection :title="$t('message.faderPaging.general')" :options="getEnabledOptions()"
                        @selection-changed="onEnabledChanged"/>
        <RadioSelection :title="$t('message.faderPaging.pageTitle')" group="fader_paging_bank"
                        :options="getBankOptions()" :selected="activeBank"
                        @selection-changed="id => activeBank = id"/>
        <div class="tools">
          <button @click="captureBank">{{ $t('message.faderPaging.capture') }}</button>
          <button @click="applyBank">{{ $t('message.faderPaging.apply') }}</button>
        </div>
      </GroupContainer>

      <GroupContainer :title="$t('message.faderPaging.faderTitle', { bank: activeBank })">
        <RadioSelection :title="$t('message.faderPaging.faderSelect')" group="fader_paging_fader"
                        :options="getFaderOptions()" :selected="activeFader"
                        @selection-changed="id => activeFader = id"/>
        <RadioSelection :title="$t('message.faderPaging.channel')" group="fader_paging_channel"
                        :options="getChannelOptions()" :selected="getChannel()"
                        @selection-changed="onChannelChanged"/>
        <RadioSelection :title="$t('message.configuration.muteBehaviourTitle')" group="fader_paging_mute"
                        :options="getMuteFunctionOptions()" :selected="getMuteFunction()"
                        @selection-changed="onMuteFunctionChanged"/>
        <div class="tools">
          <button @click="clearEntry">{{ $t('message.faderPaging.clear') }}</button>
        </div>
      </GroupContainer>

      <GroupContainer :title="$t('message.lighting.mixer.fader.title')">
        <CheckSelection :title="$t('message.lighting.mixer.fader.style')" :options="getFaderLightingOptions()"
                        @selection-changed="onFaderLightingChanged"/>
        <template v-if="hasFaderColours()">
          <ColourPicker :title="$t('message.lighting.mixer.fader.bottomColour')" :color-value="getFaderColour(1)"
                        @colour-changed="v => onFaderColourChanged(1, v)"/>
          <ColourPicker :title="$t('message.lighting.mixer.fader.topColour')" :color-value="getFaderColour(0)"
                        @colour-changed="v => onFaderColourChanged(0, v)"/>
        </template>
      </GroupContainer>

      <GroupContainer :title="$t('message.lighting.mixer.screen.title')">
        <CheckSelection :title="$t('message.lighting.mixer.screen.optionsTitle')" :options="getScribbleOptions()"
                        @selection-changed="onScribbleOptionsChanged">
          <div v-if="hasScribble()" style="text-align: left; padding: 14px 8px 0">
            <div style="color: #fff; margin-bottom: 8px">{{ $t('message.lighting.mixer.screen.options.text') }}:</div>
            <input type="text" v-model="textValue" @blur="onScribbleTextApplied"
                   v-on:keyup.enter="onScribbleTextApplied"
                   :aria-label="$t('message.lighting.mixer.screen.options.text')"
                   :placeholder="$t('message.faderPaging.autoText')"/>
          </div>
        </CheckSelection>
        <template v-if="hasScribble()">
          <ColourPicker :title="$t('message.lighting.mixer.screen.backgroundColour')"
                        :color-value="getScribbleColour()" @colour-changed="onScribbleColourChanged"/>
          <RadioSelection group="fader_paging_icon" :title="$t('message.lighting.mixer.screen.icons')"
                          :options="getIconOptions()" :selected="getScribbleIcon()"
                          @selection-changed="onScribbleIconChanged" max-width="200px"/>
        </template>
      </GroupContainer>

      <GroupContainer :title="$t('message.lighting.mixer.mute.title')">
        <CheckSelection :title="$t('message.faderPaging.muteLighting')" :options="getMuteLightingOptions()"
                        @selection-changed="onMuteLightingChanged"/>
        <template v-if="hasMuteColours()">
          <ColourPicker :title="$t('message.lighting.common.activeColour')" :color-value="getMuteColour(0)"
                        @colour-changed="v => onMuteColourChanged(0, v)"/>
          <RadioSelection :title="$t('message.lighting.common.inactiveOption')" group="fader_paging_mute_off"
                          :options="getLightingInactiveOptions($t)" :selected="getMuteOffStyle()"
                          @selection-changed="onMuteOffStyleChanged"/>
          <ColourPicker :title="$t('message.lighting.common.inactiveColour')" :color-value="getMuteColour(1)"
                        @colour-changed="v => onMuteColourChanged(1, v)"/>
        </template>
      </GroupContainer>
    </div>
  </div>
</template>

<style scoped>
input[type=text] {
  font-family: LeagueMonoCondensed, sans-serif;
  background-color: #3b413f;
  color: #59b1b6;
  box-sizing: border-box;
  text-align: center;
  padding: 10px;
  border: none;
  outline: none;
}

.tools {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px;
}

.tools button {
  background-color: #3b413f;
  color: #fff;
  border: 0;
  padding: 8px 14px;
  cursor: pointer;
}

.tools button:hover {
  background-color: #49514e;
}
</style>
