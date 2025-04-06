<template>
   <div class="container"
         @dragover="dragoverGlobal"
         @dragleave.stop="unsetDragTarget">
      v.0.5
      <div class="current-drug-container">
         <div class="drug-choice">
            <div class="drug"
                  v-for="drug in drugs">
               <div class="button"
                     @click="selectDrug(drug)">
                  <div class="icon">
                     <img class="icon-img"
                           :src="getImg(drug.img)" />
                     <div class="icon-title">
                        {{ drug.name }}
                     </div>
                  </div>
               </div>
            </div>
         </div>
         <div class="current-drug target"
               v-if="currentProduct"
               @dragenter.stop="setDragTarget">
            <div class="name target">
               {{ currentProduct.name }}
            </div>
            <div class="image target">
               <img class="target meth"
                     :style="{
                        filter: `hue-rotate(${rand}deg) saturate(3.5)`
                     }"
                     :src="getImg(currentProduct.img)" />
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
      <div class="liked-by">
         <div class="expander"
               @click="expandMe">
            {{ customersWhoLikeCurrentEffects.length }} {{ customersWhoLikeCurrentEffects.length === 0 || customersWhoLikeCurrentEffects.length > 1 ? "People like" : "Person likes" }} this blend.
         </div>
         <div class="expand-me"
               :class="{ open: openExpander }">
            <div class="customer"
                  v-for="customer in customersWhoLikeCurrentEffects">
               <div class="customer-name">
                  <div class="customer-img">
                     <img :src="getImg(`${customer.img}`)" />
                     <div class="person-name">
                        {{ customer.name }}
                     </div>
                  </div>
               </div>
               <div class="area"
                     :class="customer.area">
                  {{ customer.area }}
               </div>
               <div class="likes">
                  Likes ({{ getLikeCount(customer.likes) }}/3):
                  <div class="like"
                        :class="{ liked: like.matched, all: (getLikeCount(customer.likes) === 3) }, styledAttribute(like.effect)"
                        v-for="like in customer.likes">
                     {{ like.effect }}
                  </div>
               </div>
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
import attributeList from "./attributes";

import additives from "./data/additives.json";
import names from "./data/names.json";
import customers from "./data/customers.json";
import drugs from "./data/drugs.json";

const images = require.context('@/assets/images/', false, /\.png$|\.jpg$/)

export default {
   data() {
      return {
         drugs: null,
         openExpander: false,
         names: {},
         customers: [],
         rand: 0,
         attributeCache: "",
         isMobile: false,
         history: [],
         attributeList: {},
         basePrice: 0,
         isCorrectTarget: false,
         currentAdditive: null,
         currentProduct: {
            name: "",
            attributes: [],
            img: "",
         }
      }
   },
   computed: {
      customersWhoLikeCurrentEffects() {
         const cstmrs = customers.filter(customer => {
            return customer.likes.some(like => {
               if (!this.currentProduct.attributes || !this.currentProduct.attributes.length) return false;
               return this.currentProduct.attributes.some(effect => {
                  if (effect === like.effect) {
                     like.matched = true;
                  }
                  return effect === like.effect
               });
            });
         });
         cstmrs.forEach(customer => {
            customer.likeCount = this.getLikeCount(customer.likes);
         });
         return cstmrs.sort((a, b) => b.likeCount - a.likeCount);
      },
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
      selectDrug(drug) {
         this.history = [];
         this.basePrice = drug.baseValue;
         this.currentProduct = {
            name: drug.name,
            attributes: drug.baseEffects,
            img: drug.img
         }
      },
      getLikeCount(likes) {
         let count = 0;
         likes.forEach(like => {
            if (like.matched) count++;
         });
         return count;
      },
      expandMe() {
         this.openExpander = !this.openExpander;
      },
      generateRandomName() {
         const prefix = this.randInt(0, 1462);
         const suffix = this.randInt(0, 131);
         const prefixes = this.names.prefixes.map(a => a);
         const suffixes = this.names.suffixes.map(a => a);
         this.currentProduct.name = `${this.capitaliseNames(prefixes[prefix])} ${this.capitaliseNames(suffixes[suffix])}`;
      },
      capitaliseNames(str) {
         if (str.includes("-")) {
            let splitString = str.split("-");
            return `${this.capitaliseFirstLetter(splitString[0])}-${this.capitaliseFirstLetter(splitString[1])}`;
         }
         return this.capitaliseFirstLetter(str);
      },
      capitaliseFirstLetter(str) {
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
         return att.replace(" ", "");
      },
      getImg(src) {
         if (!src.length) return "";
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
      this.drugs = drugs;
      this.customers = customers.map(customer => {
         customer.likes = [
            {
               effect: customer.likes[0],
               matched: false,
               style: customer.likes[0].replace(" ", "")
            }, {
               effect: customer.likes[1],
               matched: false,
               style: customer.likes[1].replace(" ", "")
            }, {
               effect: customer.likes[2],
               matched: false,
               style: customer.likes[2].replace(" ", "")
            }];
      });
      this.selectDrug(this.drugs[0]);
      if (/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) {
         this.isMobile = true;
      }
   }
}
</script>

<style src="./App.css"></style>
