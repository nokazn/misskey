<template>
<form class="eppvobhk" :class="{ signing, totpLogin }" @submit.prevent="onSubmit">
	<div class="avatar" :style="{ backgroundImage: user ? `url('${ user.avatarUrl }')` : null }" v-show="withAvatar"></div>
	<div class="normal-signin" v-if="!totpLogin">
		<mk-input v-model="username" type="text" pattern="^[a-zA-Z0-9_]+$" spellcheck="false" autofocus required @input="onUsernameChange">
			<span>{{ $t('username') }}</span>
			<template #prefix>@</template>
			<template #suffix>@{{ host }}</template>
		</mk-input>
		<mk-input v-model="password" type="password" :with-password-toggle="true" v-if="!user || user && !user.usePasswordLessLogin" required>
			<span>{{ $t('password') }}</span>
			<template #prefix><fa :icon="faLock"/></template>
		</mk-input>
		<mk-button type="submit" primary :disabled="signing" style="margin: 0 auto;">{{ signing ? $t('loggingIn') : $t('login') }}</mk-button>
		<p v-if="meta && meta.enableTwitterIntegration" style="margin: 8px 0;"><a :href="`${apiUrl}/signin/twitter`"><fa :icon="faTwitter"/> {{ $t('signinWith', { x: 'Twitter' }) }}</a></p>
		<p v-if="meta && meta.enableGithubIntegration"  style="margin: 8px 0;"><a :href="`${apiUrl}/signin/github`"><fa :icon="faGithub"/> {{ $t('signinWith', { x: 'GitHub' }) }}</a></p>
		<p v-if="meta && meta.enableDiscordIntegration" style="margin: 8px 0;"><a :href="`${apiUrl}/signin/discord`"><fa :icon="faDiscord"/> {{ $t('signinWith', { x: 'Discord' }) }}</a></p>
	</div>
	<div class="2fa-signin" v-if="totpLogin" :class="{ securityKeys: user && user.securityKeys }">
		<div v-if="user && user.securityKeys" class="twofa-group tap-group">
			<p>{{ $t('tapSecurityKey') }}</p>
			<mk-button @click="queryKey" v-if="!queryingKey">
				{{ $t('retry') }}
			</mk-button>
		</div>
		<div class="or-hr" v-if="user && user.securityKeys">
			<p class="or-msg">{{ $t('or') }}</p>
		</div>
		<div class="twofa-group totp-group">
			<p style="margin-bottom:0;">{{ $t('twoStepAuthentication') }}</p>
			<mk-input v-model="password" type="password" :with-password-toggle="true" v-if="user && user.usePasswordLessLogin" required>
				<span>{{ $t('password') }}</span>
				<template #prefix><fa :icon="faLock"/></template>
			</mk-input>
			<mk-input v-model="token" type="text" pattern="^[0-9]{6}$" autocomplete="off" spellcheck="false" required>
				<span>{{ $t('token') }}</span>
				<template #prefix><fa :icon="faGavel"/></template>
			</mk-input>
			<mk-button type="submit" :disabled="signing" primary style="margin: 0 auto;">{{ signing ? $t('loggingIn') : $t('login') }}</mk-button>
		</div>
	</div>
</form>
</template>

<script lang="ts">
import Vue from 'vue';
import { toUnicode } from 'punycode';
import { faLock, faGavel } from '@fortawesome/free-solid-svg-icons';
import { faTwitter, faDiscord, faGithub } from '@fortawesome/free-brands-svg-icons';
import MkButton from './ui/button.vue';
import MkInput from './ui/input.vue';
import i18n from '../i18n';
import { apiUrl, host } from '../config';
import { hexifyAB } from '../scripts/2fa';

export default Vue.extend({
	i18n,

	components: {
		MkButton,
		MkInput,
	},

	props: {
		withAvatar: {
			type: Boolean,
			required: false,
			default: true
		},
		autoSet: {
			type: Boolean,
			required: false,
			default: false,
		}
	},

	data() {
		return {
			signing: false,
			user: null,
			username: '',
			password: '',
			token: '',
			apiUrl,
			host: toUnicode(host),
			totpLogin: false,
			credential: null,
			challengeData: null,
			queryingKey: false,
			faLock, faGavel, faTwitter, faDiscord, faGithub
		};
	},

	computed: {
		meta() {
			return this.$store.state.instance.meta;
		},
	},

	created() {
		if (this.autoSet) {
			this.$once('login', res => {
				localStorage.setItem('i', res.i);
				location.reload();
			});
		}
	},

	methods: {
		onUsernameChange() {
			this.$root.api('users/show', {
				username: this.username
			}).then(user => {
				this.user = user;
			}, () => {
				this.user = null;
			});
		},

		queryKey() {
			this.queryingKey = true;
			return navigator.credentials.get({
				publicKey: {
					challenge: Buffer.from(
						this.challengeData.challenge
							.replace(/\-/g, '+')
							.replace(/_/g, '/'),
							'base64'
					),
					allowCredentials: this.challengeData.securityKeys.map(key => ({
						id: Buffer.from(key.id, 'hex'),
						type: 'public-key',
						transports: ['usb', 'nfc', 'ble', 'internal']
					})),
					timeout: 60 * 1000
				}
			}).catch(() => {
				this.queryingKey = false;
				return Promise.reject(null);
			}).then(credential => {
				this.queryingKey = false;
				this.signing = true;
				return this.$root.api('signin', {
					username: this.username,
					password: this.password,
					signature: hexifyAB(credential.response.signature),
					authenticatorData: hexifyAB(credential.response.authenticatorData),
					clientDataJSON: hexifyAB(credential.response.clientDataJSON),
					credentialId: credential.id,
					challengeId: this.challengeData.challengeId
				});
			}).then(res => {
				this.$emit('login', res);
			}).catch(err => {
				if (err === null) return;
				this.$root.dialog({
					type: 'error',
					text: this.$t('login-failed')
				});
				this.signing = false;
			});
		},

		onSubmit() {
			this.signing = true;
			if (!this.totpLogin && this.user && this.user.twoFactorEnabled) {
				if (window.PublicKeyCredential && this.user.securityKeys) {
					this.$root.api('signin', {
						username: this.username,
						password: this.password
					}).then(res => {
						this.totpLogin = true;
						this.signing = false;
						this.challengeData = res;
						return this.queryKey();
					}).catch(() => {
						this.$root.dialog({
							type: 'error',
							text: this.$t('login-failed')
						});
						this.challengeData = null;
						this.totpLogin = false;
						this.signing = false;
					});
				} else {
					this.totpLogin = true;
					this.signing = false;
				}
			} else {
				this.$root.api('signin', {
					username: this.username,
					password: this.password,
					token: this.user && this.user.twoFactorEnabled ? this.token : undefined
				}).then(res => {
					this.$emit('login', res);
				}).catch(() => {
					this.$root.dialog({
						type: 'error',
						text: this.$t('loginFailed')
					});
					this.signing = false;
				});
			}
		}
	}
});
</script>

<style lang="scss" scoped>
.eppvobhk {
	> .avatar {
		margin: 0 auto 0 auto;
		width: 64px;
		height: 64px;
		background: #ddd;
		background-position: center;
		background-size: cover;
		border-radius: 100%;
	}
}
</style>
