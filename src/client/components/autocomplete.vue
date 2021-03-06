<template>
<div class="swhvrteh" @contextmenu.prevent="() => {}">
	<ol class="hashtags" ref="suggests" v-if="hashtags.length > 0">
		<li v-for="hashtag in hashtags" :key="hashtag" @click="complete(type, hashtag)" @keydown="onKeydown" tabindex="-1">
			<span class="name">{{ hashtag }}</span>
		</li>
	</ol>
	<ol class="emojis" ref="suggests" v-if="emojis.length > 0">
		<li v-for="(emoji, i) in emojis" :key="i" @click="complete(type, emoji.emoji)" @keydown="onKeydown" tabindex="-1">
			<span class="emoji" v-if="emoji.isCustomEmoji"><img :src="$store.state.device.disableShowingAnimatedImages ? getStaticImageUrl(emoji.url) : emoji.url" :alt="emoji.emoji"/></span>
			<span class="emoji" v-else-if="!useOsNativeEmojis"><img :src="emoji.url" :alt="emoji.emoji"/></span>
			<span class="emoji" v-else>{{ emoji.emoji }}</span>
			<span class="name" v-html="emoji.name.replace(q, `<b>${q}</b>`)"></span>
			<span class="alias" v-if="emoji.aliasOf">({{ emoji.aliasOf }})</span>
		</li>
	</ol>
</div>
</template>

<script lang="ts">
import Vue from 'vue';
import { emojilist } from '../../misc/emojilist';
import contains from '../scripts/contains';
import { twemojiSvgBase } from '../../misc/twemoji-base';
import { getStaticImageUrl } from '../scripts/get-static-image-url';

type EmojiDef = {
	emoji: string;
	name: string;
	aliasOf?: string;
	url?: string;
	isCustomEmoji?: boolean;
};

const lib = emojilist.filter(x => x.category !== 'flags');

const char2file = (char: string) => {
	let codes = Array.from(char).map(x => x.codePointAt(0).toString(16));
	if (!codes.includes('200d')) codes = codes.filter(x => x != 'fe0f');
	codes = codes.filter(x => x && x.length);
	return codes.join('-');
};

const emjdb: EmojiDef[] = lib.map(x => ({
	emoji: x.char,
	name: x.name,
	aliasOf: null,
	url: `${twemojiSvgBase}/${char2file(x.char)}.svg`
}));

for (const x of lib) {
	if (x.keywords) {
		for (const k of x.keywords) {
			emjdb.push({
				emoji: x.char,
				name: k,
				aliasOf: x.name,
				url: `${twemojiSvgBase}/${char2file(x.char)}.svg`
			});
		}
	}
}

emjdb.sort((a, b) => a.name.length - b.name.length);

export default Vue.extend({
	props: {
		type: {
			type: String,
			required: true,
		},

		q: {
			type: String,
			required: false,
		},

		textarea: {
			type: HTMLTextAreaElement,
			required: true,
		},

		complete: {
			type: Function,
			required: true,
		},

		close: {
			type: Function,
			required: true,
		},

		x: {
			type: Number,
			required: true,
		},

		y: {
			type: Number,
			required: true,
		},
	},

	data() {
		return {
			getStaticImageUrl,
			fetching: true,
			hashtags: [],
			emojis: [],
			select: -1,
			emojilist,
			emojiDb: [] as EmojiDef[]
		}
	},

	computed: {
		items(): HTMLCollection {
			return (this.$refs.suggests as Element).children;
		},

		useOsNativeEmojis(): boolean {
			return this.$store.state.device.useOsNativeEmojis;
		}
	},

	updated() {
		this.setPosition();
	},

	mounted() {
		this.setPosition();

		//#region Construct Emoji DB
		const customEmojis = this.$store.state.instance.meta.emojis;
		const emojiDefinitions: EmojiDef[] = [];

		for (const x of customEmojis) {
			emojiDefinitions.push({
				name: x.name,
				emoji: `:${x.name}:`,
				url: x.url,
				isCustomEmoji: true
			});

			if (x.aliases) {
				for (const alias of x.aliases) {
					emojiDefinitions.push({
						name: alias,
						aliasOf: x.name,
						emoji: `:${x.name}:`,
						url: x.url,
						isCustomEmoji: true
					});
				}
			}
		}

		emojiDefinitions.sort((a, b) => a.name.length - b.name.length);

		this.emojiDb = emojiDefinitions.concat(emjdb);
		//#endregion

		this.textarea.addEventListener('keydown', this.onKeydown);

		for (const el of Array.from(document.querySelectorAll('body *'))) {
			el.addEventListener('mousedown', this.onMousedown);
		}

		this.$nextTick(() => {
			this.exec();

			this.$watch('q', () => {
				this.$nextTick(() => {
					this.exec();
				});
			});
		});
	},

	beforeDestroy() {
		this.textarea.removeEventListener('keydown', this.onKeydown);

		for (const el of Array.from(document.querySelectorAll('body *'))) {
			el.removeEventListener('mousedown', this.onMousedown);
		}
	},

	methods: {
		setPosition() {
			if (this.x + this.$el.offsetWidth > window.innerWidth) {
				this.$el.style.left = (window.innerWidth - this.$el.offsetWidth) + 'px';
			} else {
				this.$el.style.left = this.x + 'px';
			}

			if (this.y + this.$el.offsetHeight > window.innerHeight) {
				this.$el.style.top = (this.y - this.$el.offsetHeight) + 'px';
				this.$el.style.marginTop = '0';
			} else {
				this.$el.style.top = this.y + 'px';
				this.$el.style.marginTop = 'calc(1em + 8px)';
			}
		},

		exec() {
			this.select = -1;
			if (this.$refs.suggests) {
				for (const el of Array.from(this.items)) {
					el.removeAttribute('data-selected');
				}
			}

			if (this.type == 'hashtag') {
				if (this.q == null || this.q == '') {
					this.hashtags = JSON.parse(localStorage.getItem('hashtags') || '[]');
					this.fetching = false;
				} else {
					const cacheKey = `autocomplete:hashtag:${this.q}`;
					const cache = sessionStorage.getItem(cacheKey);
					if (cache) {
						const hashtags = JSON.parse(cache);
						this.hashtags = hashtags;
						this.fetching = false;
					} else {
						this.$root.api('hashtags/search', {
							query: this.q,
							limit: 30
						}).then(hashtags => {
							this.hashtags = hashtags;
							this.fetching = false;

							// キャッシュ
							sessionStorage.setItem(cacheKey, JSON.stringify(hashtags));
						});
					}
				}
			} else if (this.type == 'emoji') {
				if (this.q == null || this.q == '') {
					this.emojis = this.emojiDb.filter(x => x.isCustomEmoji && !x.aliasOf).sort((a, b) => {
						var textA = a.name.toUpperCase();
						var textB = b.name.toUpperCase();
						return (textA < textB) ? -1 : (textA > textB) ? 1 : 0;
					});
					return;
				}

				const matched = [];
				const max = 30;

				this.emojiDb.some(x => {
					if (x.name.startsWith(this.q) && !x.aliasOf && !matched.some(y => y.emoji == x.emoji)) matched.push(x);
					return matched.length == max;
				});
				if (matched.length < max) {
					this.emojiDb.some(x => {
						if (x.name.startsWith(this.q) && !matched.some(y => y.emoji == x.emoji)) matched.push(x);
						return matched.length == max;
					});
				}
				if (matched.length < max) {
					this.emojiDb.some(x => {
						if (x.name.includes(this.q) && !matched.some(y => y.emoji == x.emoji)) matched.push(x);
						return matched.length == max;
					});
				}

				this.emojis = matched;
			}
		},

		onMousedown(e) {
			if (!contains(this.$el, e.target) && (this.$el != e.target)) this.close();
		},

		onKeydown(e) {
			const cancel = () => {
				e.preventDefault();
				e.stopPropagation();
			};

			switch (e.which) {
				case 10: // [ENTER]
				case 13: // [ENTER]
					if (this.select !== -1) {
						cancel();
						(this.items[this.select] as any).click();
					} else {
						this.close();
					}
					break;

				case 27: // [ESC]
					cancel();
					this.close();
					break;

				case 38: // [↑]
					if (this.select !== -1) {
						cancel();
						this.selectPrev();
					} else {
						this.close();
					}
					break;

				case 9: // [TAB]
				case 40: // [↓]
					cancel();
					this.selectNext();
					break;

				default:
					e.stopPropagation();
					this.textarea.focus();
			}
		},

		selectNext() {
			if (++this.select >= this.items.length) this.select = 0;
			this.applySelect();
		},

		selectPrev() {
			if (--this.select < 0) this.select = this.items.length - 1;
			this.applySelect();
		},

		applySelect() {
			for (const el of Array.from(this.items)) {
				el.removeAttribute('data-selected');
			}

			this.items[this.select].setAttribute('data-selected', 'true');
			(this.items[this.select] as any).focus();
		},
	}
});
</script>

<style lang="scss" scoped>
.swhvrteh {
	position: fixed;
	z-index: 65535;
	max-width: 100%;
	margin-top: calc(1em + 8px);
	overflow: hidden;
	background: var(--panel);
	border: solid 1px rgba(#000, 0.1);
	border-radius: 4px;
	transition: top 0.1s ease, left 0.1s ease;

	> ol {
		display: block;
		margin: 0;
		padding: 4px 0;
		max-height: 190px;
		max-width: 500px;
		overflow: auto;
		list-style: none;

		> li {
			display: flex;
			align-items: center;
			padding: 4px 12px;
			white-space: nowrap;
			overflow: hidden;
			font-size: 0.9em;
			cursor: default;

			&, * {
				user-select: none;
			}

			* {
				overflow: hidden;
				text-overflow: ellipsis;
			}

			&:hover {
				background: var(--yrnqrguo);
			}

			&[data-selected='true'] {
				background: var(--accent);

				&, * {
					color: #fff !important;
				}
			}

			&:active {
				background: var(--accentDarken);

				&, * {
					color: #fff !important;
				}
			}
		}
	}

	> .emojis > li {

		.emoji {
			display: inline-block;
			margin: 0 4px 0 0;
			width: 24px;

			> img {
				width: 24px;
				vertical-align: bottom;
			}
		}

		.alias {
			margin: 0 0 0 8px;
		}
	}
}
</style>
