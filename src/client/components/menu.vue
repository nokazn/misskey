<template>
<x-popup :source="source" :no-center="noCenter" :fixed="fixed" :width="width" ref="popup" @closed="() => { $emit('closed'); destroyDom(); }" v-hotkey.global="keymap">
	<sequential-entrance class="rrevdjwt" :class="{ left: align === 'left' }" :delay="15" :direction="direction" ref="items">
		<template v-for="(item, i) in items.filter(item => item !== undefined)">
			<div v-if="item === null" class="divider" :key="i"></div>
			<span v-else-if="item.type === 'label'" class="label item" :key="i">
				<span>{{ item.text }}</span>
			</span>
			<router-link v-else-if="item.type === 'link'" :to="item.to" @click.native="close()" :tabindex="i" class="_button item" :key="i">
				<fa v-if="item.icon" :icon="item.icon" fixed-width/>
				<mk-avatar v-if="item.avatar" :user="item.avatar" class="avatar"/>
				<span>{{ item.text }}</span>
				<i v-if="item.indicate"><fa :icon="faCircle"/></i>
			</router-link>
			<a v-else-if="item.type === 'a'" :href="item.href" :target="item.target" :download="item.download" @click="close()" :tabindex="i" class="_button item" :key="i">
				<fa v-if="item.icon" :icon="item.icon" fixed-width/>
				<span>{{ item.text }}</span>
				<i v-if="item.indicate"><fa :icon="faCircle"/></i>
			</a>
			<button v-else-if="item.type === 'user'" @click="clicked(item.action)" :tabindex="i" class="_button item" :key="i">
				<mk-avatar :user="item.user" class="avatar"/><mk-user-name :user="item.user"/>
				<i v-if="item.indicate"><fa :icon="faCircle"/></i>
			</button>
			<button v-else @click="clicked(item.action)" :tabindex="i" class="_button item" :key="i">
				<fa v-if="item.icon" :icon="item.icon" fixed-width/>
				<mk-avatar v-if="item.avatar" :user="item.avatar" class="avatar"/>
				<span>{{ item.text }}</span>
				<i v-if="item.indicate"><fa :icon="faCircle"/></i>
			</button>
		</template>
	</sequential-entrance>
</x-popup>
</template>

<script lang="ts">
import Vue from 'vue';
import { faCircle } from '@fortawesome/free-solid-svg-icons';
import XPopup from './popup.vue';
import { focusPrev, focusNext } from '../scripts/focus';

export default Vue.extend({
	components: {
		XPopup
	},
	props: {
		source: {
			required: true
		},
		items: {
			type: Array,
			required: true
		},
		align: {
			type: String,
			required: false
		},
		noCenter: {
			type: Boolean,
			required: false
		},
		fixed: {
			type: Boolean,
			required: false
		},
		width: {
			type: Number,
			required: false
		},
		direction: {
			type: String,
			required: false
		},
		viaKeyboard: {
			type: Boolean,
			required: false
		},
	},
	data() {
		return {
			faCircle
		};
	},
	computed: {
		keymap(): any {
			return {
				'up|k|shift+tab': this.focusUp,
				'down|j|tab': this.focusDown,
			};
		},
	},
	mounted() {
		if (this.viaKeyboard) {
			this.$nextTick(() => {
				focusNext(this.$refs.items.$slots.default[0].elm, true);
			});
		}
	},
	methods: {
		clicked(fn) {
			fn();
			this.close();
		},
		close() {
			this.$refs.popup.close();
		},
		focusUp() {
			focusPrev(document.activeElement);
		},
		focusDown() {
			focusNext(document.activeElement);
		}
	}
});
</script>

<style lang="scss" scoped>
.rrevdjwt {
	padding: 8px 0;

	&.left {
		> .item {
			text-align: left;
		}
	}

	> .item {
		display: block;
		position: relative;
		padding: 8px 16px;
		width: 100%;
		box-sizing: border-box;
		white-space: nowrap;
		font-size: 0.9em;
		line-height: 20px;
		text-align: center;
		overflow: hidden;
		text-overflow: ellipsis;

		&:hover {
			color: #fff;
			background: var(--accent);
			text-decoration: none;
		}

		&:active {
			color: #fff;
			background: var(--accentDarken);
		}

		&:not(:active):focus {
			box-shadow: 0 0 0 2px var(--focus) inset;
		}

		&.label {
			pointer-events: none;
			font-size: 0.7em;
			padding-bottom: 4px;

			> span {
				opacity: 0.7;
			}
		}

		> [data-icon] {
			margin-right: 4px;
			width: 20px;
		}

		> .avatar {
			margin-right: 4px;
			width: 20px;
			height: 20px;
		}

		> i {
			position: absolute;
			top: 5px;
			left: 13px;
			color: var(--indicator);
			font-size: 12px;
			animation: blink 1s infinite;
		}
	}

	> .divider {
		margin: 8px 0;
		height: 1px;
		background: var(--divider);
	}
}
</style>
