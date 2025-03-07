<template>
  <div class="card">
    <v-expansion-panels v-model="panel">
      <v-expansion-panel>
        <v-expansion-panel-header>
          <div class="handle" style="font-size: 20px;">
            <v-icon
              style="margin-right: 10px;"
              @click="onClose"
            >close</v-icon>
            <span>InstanceList</span>
          </div>
        </v-expansion-panel-header>
        <v-expansion-panel-content>
          <v-switch v-model="show_predictions" class="mx-2" label="predictions"></v-switch>
          <v-switch v-model="show_features" class="mx-2" label="features"></v-switch>
          <v-slider
            v-model="string_length"
            min="1"
            max="100"
            label="string length"
            thumb-label
          ></v-slider>
          <v-radio-group label="sort by selection" v-model="sortBySel">
            <v-radio
              key="none"
              label="none"
              value=""
            ></v-radio>
            <v-radio
              key="first"
              label="first"
              value="first"
            ></v-radio>
            <v-radio
              key="second"
              label="second"
              value="second"
            ></v-radio>
            <v-radio
              key="common"
              label="common"
              value="common"
            ></v-radio>
          </v-radio-group>
        </v-expansion-panel-content>
      </v-expansion-panel>
    </v-expansion-panels>
    <v-data-table
      dense
      :headers="headers"
      :items="instances"
      class="elevation-1"
      :sort-by.sync="sortby"
      :sort-desc.sync="descending"
    >
    <template v-slot:body="{ items }">
      <tbody>
        <tr
          v-for="item in items"
          :key="item.id"
          @mouseenter="changedHoverInstance(item.id)"
          @mouseleave="clearHoverInstance"
          @click.stop.prevent="click(item)"
          @contextmenu.stop.prevent="select(item.id, 'second')"
          :class="{'hover-instance': item === $store.state.hoverInstance}"
          :style="styleTableRow(item)"
        >
          <td class="unselectable">{{ item.id }}</td>
          <td class="unselectable">{{ item.actual }}</td>
          <td  
            class="unselectable"
            v-for="classifier in continuousClassifierList"
            :key="item.id+classifier+'-'+item.continuous_predictions[classifier]"
          >
            {{ item.continuous_predictions[classifier] }}
          </td>
          <td
            class="unselectable"
            v-for="classifier in classifiers"
            :key="item.id+classifier+'-'+item.predictions[classifier]"
          >
            {{ item.predictions[classifier] }}
          </td>
          <td
            class="unselectable"
            v-for="feature in features"
            :key="item.id+feature+'-'+item.actual"
            :title="item.features[feature]"
          >
            {{ shorten(item.features[feature]) }}
          </td>
        </tr>
      </tbody>
    </template>
    </v-data-table>
  </div>
</template>

<script lang="ts">
import { colors } from '../../theme';
import {
  Instance,
  InstanceWithId,
  SelectionRecord,
  BoxProps,
} from '../../types';
import { instanceById } from '../../utils';
import Vue from 'vue';
import { mapActions } from 'vuex';
import { color } from 'd3';
interface Header {
  text: string;
  value: string;
}
export default Vue.extend({
  name: 'InstanceList',
  props: {
    boxProps: {
      type: Object as () => BoxProps,
      required: true,
    },
    onClose: {
      type: Function,
      required: true,
    },
  },
  data() {
    return {
      panel: [],
      sortby: [],
      sortBySel: '' as '' | 'first' | 'second' | 'common',
      descending: false,
      show_predictions: true,
      show_features: true,
      string_length: 7,
      clickCount: 0,
      clickTimer: 0,
    };
  },
  computed: {
    continuousClassifierList: function () {
      if (this.datasetType == "continuous") {
        return this.classifiers
      }
      return []
    },
    datasetType(): string{
      var datasetType = this.$store.state.datasetType;
      return datasetType;
    },
    classifiers(): string[] {
      const dummyClassifiers = [
        'Oracle',
        'Majority Classifier',
        'Random Classifier',
      ];
      if (this.show_predictions) {
        const classifiers = [...this.$store.getters.filteredClassifiers];
        if (this.$store.state.datasetType == "continuous") {
          return classifiers.filter((c) => !dummyClassifiers.includes(c))
        }
        return classifiers;
      }
      return [ ];
    },
    features(): string[] {
      if (this.show_features) {
        const features = [...this.$store.getters.features];
        return features;
      }
      return [ ];
    },
    headers(): Header[] {
      const headers = [
        { text: 'Id', value: 'id' },
        { text: 'Actual class', value: 'actualClass' },
      ];
      if (this.show_predictions && this.$store.state.datasetType == "continuous") {
        this.classifiers.forEach((classifier: string) => {
          headers.push({
            text: classifier.replace(/_/g, ' ')+"_"+"value",
            value: classifier,
          });
        });
      }
      if (this.show_predictions) {
        this.classifiers.forEach((classifier: string) => {
          headers.push({
            text: classifier.replace(/_/g, ' '),
            value: classifier,
          });
        });
      }
      if (this.show_features) {
        this.$store.getters.features.forEach((feature: string) => {
          headers.push({
            text: feature,
            value: feature,
          });
        });
      }
      return headers;
    },
    instances(): InstanceWithId[] {
      const instances: InstanceWithId[] = [];
      this.$store.getters.filteredInstances.forEach((id: string) => {
        const i = instanceById(id);
        const instance: InstanceWithId = {
          id,
          ...i,
        };
        instances.push(instance);
      });
      const { first, second } = this.selections;
      const comparators = {
        first: (a: InstanceWithId, b: InstanceWithId) => {
          let aBelongs = 0;
          let bBelongs = 0;
          if (first && first.instances.has(a.id)) {
            aBelongs++;
          }
          if (first && first.instances.has(b.id)) {
            bBelongs++;
          }
          return bBelongs - aBelongs;
        },
        second: (a: InstanceWithId, b: InstanceWithId) => {
          let aBelongs = 0;
          let bBelongs = 0;
          if (second && second.instances.has(a.id)) {
            aBelongs++;
          }
          if (second && second.instances.has(b.id)) {
            bBelongs++;
          }
          return bBelongs - aBelongs;
        },
        common: (a: InstanceWithId, b: InstanceWithId) => {
          let aBelongs = 0;
          let bBelongs = 0;
          if (first && first.instances.has(a.id)) {
            aBelongs++;
          }
          if (first && first.instances.has(b.id)) {
            bBelongs++;
          }
          if (second && second.instances.has(a.id)) {
            aBelongs++;
          }
          if (second && second.instances.has(b.id)) {
            bBelongs++;
          }
          return bBelongs - aBelongs;
        },
      };
      // sort by value first, then by group
      let lsort = this.sortby[0];
      let predOrFeature = instances[0].predictions.hasOwnProperty(lsort) ? "predictions" : "features";
      if (this.sortby.length > 0) {
        instances.sort((a, b) => {
          let ret = ("" + a[predOrFeature][lsort]).localeCompare(b[predOrFeature][lsort]);
          return this.descending ? -ret : ret;
        });
      }
      if (this.sortBySel) {
        instances.sort(comparators[this.sortBySel]);
      }
      return instances;
    },
    selections(): {
      first: SelectionRecord | null,
      second: SelectionRecord | null,
    } {
      return this.$store.state.overlapSelections;
    },
  },
  watch: {
    sortby() {
      console.log("sortby ", this.sortby);
    },
    descending() {
      console.log("descending", this.descending);
    },
    selections() {
      const { first, second } = this.selections;
      if (first && second) {
        this.sortBy = 'common';
      } else if (first) {
        this.sortBy = 'first';
      } else if (second) {
        this.sortBy = 'second';
      } else {
        this.sortBy = '';
      }
    },
  },
  methods: {
    ...mapActions([
      'changedHoverInstance',
      'clearHoverInstance',
    ]),
    shorten(feature) {
      let stringFeature = "" + feature;
      if (stringFeature.length > this.string_length) {
        return stringFeature.substring(0, this.string_length) + "..."
      } else {
        return stringFeature;
      }
    },
    select(
      id: string,
      whichOverlap: 'first' | 'second'
    ) {
      const instances = new Set([id]);
      const description = `Instance #${id}`;
      this.$store.dispatch('prependToSelectionHistory', { instances, description });
      this.$store.dispatch('changedMostRecentSelection', whichOverlap);
    },
    click(item){
      this.clickCount += 1
      if (this.clickCount === 1) {
          this.clickTimer = setTimeout(function(context) {
              context.clickCount = 0
              // single click event
              context.select(item.id, 'first');
          }, 300, this)
      } 
      if (this.clickCount === 2) {
          clearTimeout(this.clickTimer);
          this.clickCount = 0
          // double click event
          this.setFocus(item.id);
      }
    },
    setFocus(id){
      this.$store.dispatch("setFocusItem", id);
    },
    styleTableRow(instance: InstanceWithId) {
      const { first: firstColor, second: secondColor } = colors.overlapSelections;
      const focusColor = colors.focus;
      const { first: firstSelection, second: secondSelection } = this.selections;
      const isFirst = (firstSelection && firstSelection.instances.has(instance.id));
      const isSecond = (secondSelection && secondSelection.instances.has(instance.id));
      const inFocus = (this.$store.state.focusItemId === instance.id)
      if(inFocus) {
        return `
          background-color: ${focusColor};
        `;
      }
      if (isFirst && isSecond) {
        return `
          background: -webkit-linear-gradient(top, ${firstColor}, ${secondColor});
          color: white;
        `;
      }
      if (isFirst) {
        return `
          background-color: ${firstColor};
        `;
      }
      if (isSecond) {
        return `
          background-color: ${secondColor};
          color: white;
        `;
      }
      return '';
    },
  },
});
</script>

<style scoped>
tr.hover-instance td {
  background-color: yellow;
}
.unselectable {
  -webkit-user-select: none; /* Safari */
  -ms-user-select: none; /* IE 10 and IE 11 */
  user-select: none; /* Standard syntax */
}
</style>