<template>
   <div class="container"
         @dragover="dragoverGlobal"
         @dragleave.stop="unsetDragTarget">
      v.0.3
      <div class="current-drug-container">
         <div class="current-drug target"
               @dragenter.stop="setDragTarget">
            <div class="name">
               {{ currentProduct.name }}
            </div>
            <div class="image target">
               <img class="target meth"
                     :style="{
                        filter: `hue-rotate(${rand}deg) saturate(3.5)`
                     }"
                     :src="getImg('meth.png')" />
            </div>
            <div class="attributes target">
               <div class="title target"
                     v-if="currentProduct.attributes.length">Attributes:</div>
               <div class="attribute-container target">
                  <div class="attribute target"
                        :class="styledAttribute(attribute)"
                        v-for="attribute in currentProduct.attributes">
                     &bull; {{ attribute }}
                  </div>
               </div>
            </div>
            <div class="price target">
               $ {{ price }}
            </div>
         </div>
         <div class="history">
            <b>Sequence:</b>
            <div class="step"
                  v-for="(step, idx) in history">
               {{ idx + 1 }}. {{ step }}
            </div>
         </div>
      </div>

      <div v-if="items"
            class="items">
         <div class="item-container"
               v-for="(add, idx) in items"
               :key="idx">
            <div class="name">
               {{ add.name }}
            </div>
            <div class="item-image">
               <div class="img-container draggable"
                     @click="applyAdditiveIfMobile(add)"
                     @dragstart="dragging(add, $event)"
                     @dragend="applyAdditive(add, $event)"
                     draggable>
                  <img class="img"
                        :src="getImg(add.image)" />
               </div>
            </div>
            <div class="effect">
               {{ add.effect }}
            </div>
            <div class="effects">
               <div class="removes">
                  <div class="remove"
                        :class="{ active: currentProduct.attributes.find(e => e === remove) }"
                        v-for="(remove, i) in add.removes"
                        :key="idx * i + 1">
                     - {{ remove }}
                  </div>
               </div>
               <div class="adds">
                  <div class="add"
                        :class="{ active: getIsActiveRemoval(i, add.removes) }"
                        v-for="(addition, i) in add.adds"
                        :key="idx * i">
                     + {{ addition }}
                  </div>
               </div>
            </div>
         </div>
      </div>
   </div>
</template>

<script>
import additives from "./data/additives.json";
import attributeList from "./attributes";
import names from "./data/names.json";
const images = require.context('@/assets/images/', false, /\.png$|\.jpg$/)

export default {
   data() {
      return {
         names: {},
         rand: 0,
         attributeCache: "",
         isMobile: false,
         history: [],
         attributeList: {},
         basePrice: 70,
         isCorrectTarget: false,
         currentAdditive: null,
         currentProduct: {
            name: "Meth",
            attributes: []
         }
      }
   },
   computed: {
      items() {
         if (!additives) return [];
         return additives.additives;
      },
      price() {
         let price = this.basePrice;
         let mods = 0;

         this.currentProduct.attributes.forEach(att => {
            mods += this.attributeList[att].modifier;
         });
         price = price * (1 + mods);

         return Math.ceil(price);
      }
   },
   methods: {
      generateRandomName() {
         const prefix = this.randInt(0, 1465);
         const suffix = this.randInt(0, 109);
         const prefixes = this.names.prefixes.map(a => a);
         const suffixes = this.names.suffixes.map(a => a);
         this.currentProduct.name = `${this.capitaliseFirstLetter(prefixes[prefix])} ${this.capitaliseFirstLetter(suffixes[suffix])}`;
      },
      capitaliseFirstLetter(str) {
         if (str.includes("-")) {
            let str1 = str.split("-")[0];
            let str2 = str.split("-")[1];

            str1 = `${str1.substring(0, 1).toUpperCase()}${str1.substring(1, str1.length)}`;
            str2 = `${str2.substring(0, 1).toUpperCase()}${str2.substring(1, str2.length)}`;
            return `${str1}-${str2}`;
         }
         return `${str.substring(0, 1).toUpperCase()}${str.substring(1, str.length)}`;
      },
      randInt(min, max) {
         return parseInt(Math.random() * (max - min) + min);
      },
      dragoverGlobal(ev) {
         ev.dataTransfer.dropEffect = "move";
         ev.preventDefault();
      },
      getIsActiveRemoval(index, removalArray) {
         return this.currentProduct.attributes.some(att => att === removalArray[index])
      },
      styledAttribute(att) {
         // plep
         return att.replace(" ", "");
      },
      getImg(src) {
         const img = images("./" + src);
         return img;
      },
      dragging(item, ev) {
         ev.dataTransfer.setData("text/plain", "");
         ev.target.classList.add("dragging");
         this.currentAdditive = item;
      },
      applyAdditiveIfMobile(add) {
         if (!this.isMobile) return;
         this.currentAdditive = add;
         this.isCorrectTarget = true;
         this.applyAdditive(add);
      },
      applyAdditive(add, event) {
         this.attributeCache = JSON.stringify(this.currentProduct.attributes);
         if (event) event.target.classList.remove("dragging");
         if (!this.isCorrectTarget) return;
         if (this.currentProduct.attributes.length) {
            this.checkForRemovables();
         }
         if (!this.currentProduct.attributes.find(item => this.currentAdditive.effect === item)) {
            if (this.currentProduct.attributes.length !== 8) {
               this.currentProduct.attributes.push(this.currentAdditive.effect);
            }
         }
         if (JSON.stringify(this.currentProduct.attributes) !== this.attributeCache) {
            this.rand = this.randInt(0, 359);
            this.generateRandomName();
         }
         this.history.push(this.currentAdditive.name);
      },
      setDragTarget() {
         this.isCorrectTarget = true;
      },
      unsetDragTarget(ev) {
         if (ev.target && [...ev.target.classList].includes("target")) {
            this.isCorrectTarget = true;
         } else {
            this.isCorrectTarget = false;
         }
      },
      checkForRemovables() {
         this.currentProduct.attributes.forEach(att => {
            this.currentAdditive.removes.some((e, idx) => {
               if (e == att) {
                  if (this.currentProduct.attributes.find(att => att === this.currentAdditive.adds[idx])) {
                     return false;
                  }
                  this.currentProduct.attributes = this.currentProduct.attributes.filter(a => a !== e);
                  this.currentProduct.attributes.push(this.currentAdditive.adds[idx]);
                  return true;
               } else {
                  return false;
               }
            });
         });
      }
   },
   mounted() {
      this.attributeList = attributeList;
      this.names = names;
      if (/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) {
         this.isMobile = true;
      }
   }
}
</script>

<style src="./App.css"></style>
