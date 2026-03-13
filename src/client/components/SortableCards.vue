<template>
<div>
  <div v-if="experimentalUI()" v-i18n>
    <label>
      <input type="checkbox" v-model="showReorder" /> Reorder Cards
    </label>
  </div>
  <div ref="sortableContainer" class="sortable-cards">
    <div ref="draggers"
         v-for="(card, index) in getSortedCards()" :key="card.name"
         :data-card-name="card.name"
         :class="{ 'dragging': Boolean(dragCard), 'touch-insert-before': touchDragTarget === card.name && !touchInsertAfter, 'touch-insert-after': touchDragTarget === card.name && touchInsertAfter }"
         draggable="true"
         @dragstart="onDragStart(card.name)"
         @dragend="onDragEnd()"
         @touchstart.passive="onTouchStart(card.name, $event)">
      <div v-if="dragCard" ref="droppers" class="drop-target" @dragover="onDragOver(card.name)"></div>
      <div ref="cardbox" class="cardbox" @click="clickMethod">
        <Card :card="card"/>
        <div v-if="showReorder" class="reorder-banners-container">
          <div class="reorder-banners-left" v-if="index > 0"></div>
          <div class="reorder-banners-right" v-if="index < cards.length - 1"></div>
        </div>
      </div>
    </div>
    <div v-if="dragCard" ref="dropend" class="drop-target" @dragover="onDragOver('end')"></div>
  </div>
</div>
</template>

<script lang="ts">
import {defineComponent} from '@/client/vue3-compat';
import Card from '@/client/components/card/Card.vue';
import {CardName} from '@/common/cards/CardName';
import {CardModel} from '@/common/models/CardModel';
import {CardOrderStorage} from '@/client/utils/CardOrderStorage';
import {getPreferences} from '@/client/utils/PreferencesManager';

type DataModel = {
  /** When true use the point-and-click reorder UI */
  showReorder: boolean;
  /** Mapping from card name to its order */
  cardOrder: {[x: string]: number};
  /** When defined, it is the name of the card being dragged. */
  dragCard: CardName | undefined;
  touchDragCard: CardName | undefined;
  touchDragTarget: CardName | undefined;
  touchInsertAfter: boolean;
  touchDropTarget: CardName | 'end' | undefined;
  touchGhost: HTMLElement | undefined;
  touchStartX: number;
  touchStartY: number;
  touchDragStarted: boolean;
};

export default defineComponent({
  name: 'SortableCards',
  components: {
    Card,
  },
  props: {
    cards: {
      type: Array as () => Array<CardModel>,
      required: true,
    },
    playerId: {
      type: String,
      required: true,
    },
  },
  data(): DataModel {
    const cache = CardOrderStorage.getCardOrder(this.playerId);
    const cardOrder: {[x: string]: number} = {};
    const keys = Object.keys(cache);
    let max = 0;
    for (const key of keys) {
      if (this.cards.find((card) => card.name === key) !== undefined) {
        cardOrder[key] = cache[key];
        max = Math.max(max, cache[key]);
      }
    }
    max++;
    for (const card of this.cards) {
      if (cardOrder[card.name] === undefined) {
        cardOrder[card.name] = max++;
      }
    }
    return {
      showReorder: false,
      cardOrder: cardOrder,
      dragCard: undefined,
      touchDragCard: undefined,
      touchDragTarget: undefined,
      touchInsertAfter: false,
      touchDropTarget: undefined,
      touchGhost: undefined,
      touchStartX: 0,
      touchStartY: 0,
      touchDragStarted: false,
    };
  },
  mounted() {
    const container = this.$refs.sortableContainer as HTMLElement;
    container.addEventListener('touchmove', this.onTouchMove, {passive: false});
    document.addEventListener('touchend', this.onTouchEnd);
  },
  beforeUnmount() {
    const container = this.$refs.sortableContainer as HTMLElement;
    container.removeEventListener('touchmove', this.onTouchMove);
    document.removeEventListener('touchend', this.onTouchEnd);
  },
  methods: {
    getSortedCards() {
      return CardOrderStorage.getOrdered(this.cardOrder, this.cards);
    },
    // --- HTML5 drag (desktop) ---
    onDragStart(source: CardName): void {
      this.dragCard = source;
    },
    onDragEnd(): void {
      this.dragCard = undefined;
    },
    onDragOver(source: CardName | 'end'): void {
      if (this.dragCard === undefined || source === this.dragCard) return;
      if (source === 'end') {
        let max = 0;
        const keys = Object.keys(this.cardOrder);
        for (const key of keys) {
          max = Math.max(max, this.cardOrder[key]);
        }
        this.cardOrder[this.dragCard] = max + 1;
      } else {
        // place it ahead of the card
        const temp = this.cardOrder[source];
        const keys = Object.keys(this.cardOrder);
        for (const key of keys) {
          if (this.cardOrder[key] >= temp) {
            this.cardOrder[key]++;
          }
        }
        this.cardOrder[this.dragCard] = temp;
      }
      CardOrderStorage.updateCardOrder(this.playerId, this.cardOrder);
    },
    // --- Touch drag (mobile) ---
    onTouchStart(cardName: CardName, event: TouchEvent): void {
      const touch = event.touches[0];
      this.touchDragCard = cardName;
      this.touchStartX = touch.clientX;
      this.touchStartY = touch.clientY;
      this.touchDragStarted = false;
      this.touchDragTarget = undefined;
      this.touchInsertAfter = false;
      this.touchDropTarget = undefined;
    },
    onTouchMove(event: TouchEvent): void {
      if (!this.touchDragCard) return;
      const touch = event.touches[0];

      if (!this.touchDragStarted) {
        const dx = touch.clientX - this.touchStartX;
        const dy = touch.clientY - this.touchStartY;
        if (Math.abs(dx) < 8 && Math.abs(dy) < 8) return;
        this.touchDragStarted = true;
        this.dragCard = this.touchDragCard;

        const container = this.$refs.sortableContainer as HTMLElement;
        const cardEl = Array.from(container.querySelectorAll('[data-card-name]'))
          .find((el) => el.getAttribute('data-card-name') === this.touchDragCard) as HTMLElement | undefined;
        if (cardEl) {
          const rect = cardEl.getBoundingClientRect();
          this.touchGhost = cardEl.cloneNode(true) as HTMLElement;
          this.touchGhost.style.cssText = `
            position: fixed;
            left: ${rect.left}px;
            top: ${rect.top}px;
            width: ${rect.width}px;
            height: ${rect.height}px;
            pointer-events: none;
            opacity: 0.7;
            z-index: 9999;
          `;
          document.body.appendChild(this.touchGhost);
        }
      }

      event.preventDefault();

      if (this.touchGhost) {
        const ghostRect = this.touchGhost.getBoundingClientRect();
        this.touchGhost.style.left = `${touch.clientX - ghostRect.width / 2}px`;
        this.touchGhost.style.top = `${touch.clientY - ghostRect.height / 2}px`;
      }

      // Find which card is under the finger (hide ghost so elementFromPoint sees through it)
      if (this.touchGhost) this.touchGhost.style.visibility = 'hidden';
      const element = document.elementFromPoint(touch.clientX, touch.clientY);
      if (this.touchGhost) this.touchGhost.style.visibility = '';

      const wrapper = element?.closest('[data-card-name]') as HTMLElement | null;
      if (wrapper) {
        const name = wrapper.getAttribute('data-card-name') as CardName;
        if (name === this.touchDragCard) {
          this.touchDragTarget = undefined;
          this.touchDropTarget = undefined;
        } else {
          this.touchDragTarget = name;
          // Left half of card → insert before; right half → insert after.
          // "Insert after X" = call onDragOver with the next card in display order.
          const wrapperRect = wrapper.getBoundingClientRect();
          const onRightHalf = touch.clientX > wrapperRect.left + wrapperRect.width / 2;
          this.touchInsertAfter = onRightHalf;
          if (onRightHalf) {
            const container = this.$refs.sortableContainer as HTMLElement;
            const allWrappers = Array.from(container.querySelectorAll('[data-card-name]'));
            const idx = allWrappers.indexOf(wrapper);
            // Skip the dragged card if it's the immediate next sibling
            const next = allWrappers[idx + 1];
            const nextName = next?.getAttribute('data-card-name') as CardName | null;
            if (!next || nextName === null) {
              this.touchDropTarget = 'end';
            } else if (nextName === this.touchDragCard) {
              const after = allWrappers[idx + 2];
              const afterName = after?.getAttribute('data-card-name') as CardName | null;
              this.touchDropTarget = afterName ?? 'end';
            } else {
              this.touchDropTarget = nextName;
            }
          } else {
            this.touchInsertAfter = false;
            this.touchDropTarget = name;
          }
        }
      } else {
        this.touchDragTarget = undefined;
        this.touchInsertAfter = false;
        const container = this.$refs.sortableContainer as HTMLElement;
        const rect = container.getBoundingClientRect();
        this.touchDropTarget = touch.clientX > rect.right ? 'end' : undefined;
      }
    },
    onTouchEnd(): void {
      if (this.touchDragStarted) {
        if (this.touchDropTarget !== undefined) {
          this.onDragOver(this.touchDropTarget);
        }
        if (this.touchGhost) {
          document.body.removeChild(this.touchGhost);
          this.touchGhost = undefined;
        }
        this.dragCard = undefined;
      }
      this.touchDragCard = undefined;
      this.touchDragTarget = undefined;
      this.touchInsertAfter = false;
      this.touchDropTarget = undefined;
      this.touchDragStarted = false;
    },
    // --- Click-to-reorder (experimental UI) ---
    clickMethod(e: MouseEvent) {
      if (!this.showReorder) return;
      const target = e.currentTarget as HTMLElement;
      if (!target) return;
      if (target.matches('.sortable-cards *')) {
        const rect = target.getBoundingClientRect();
        const x = (e.clientX - rect.left) / rect.width;
        const direction = x <= 0.25 ? -1.5 : x >= 0.75 ? 1.5 : null;
        if (direction) {
          const cardTitle = target.querySelector('.card-title');
          if (cardTitle) {
            const textContent = cardTitle.textContent;
            if (textContent) {
              const thisCard = textContent.trim();
              this.cardOrder[thisCard] += direction;
              Object.entries(this.cardOrder)
                .sort((a, b) => a[1]-b[1])
                .forEach((entry, i) => {
                  this.cardOrder[entry[0]] = i+1;
                });
              CardOrderStorage.updateCardOrder(this.playerId, this.cardOrder);
            }
          }
        }
      }
    },
    experimentalUI(): boolean {
      return getPreferences().experimental_ui;
    },
  },
});
</script>
