<template>
<x-window ref="window" :height="270" @closed="() => { $emit('closed'); destroyDom(); }">
	<template #header>{{ $t('login') }}</template>
	<mk-signin :auto-set="autoSet" @login="onLogin"/>
</x-window>
</template>

<script lang="ts">
import Vue from 'vue';
import XWindow from './window.vue';
import MkSignin from './signin.vue';

export default Vue.extend({
	components: {
		MkSignin,
		XWindow,
	},

	props: {
		autoSet: {
			type: Boolean,
			required: false,
			default: false,
		}
	},

	methods: {
		onLogin(res) {
			this.$emit('login', res);
			this.$refs.window.close();
		}
	}
});
</script>
