<template>
	<form id="sieve-form">
		<div class="flex-row">
			<input
				id="sieve-disabled"
				v-model="sieveConfig.sieveEnabled"
				type="radio"
				name="sieve-active"
				:disabled="loading"
				:value="false">
			<label class="button"
				for="sieve-disabled"
				:class="{primary: !sieveConfig.sieveEnabled}">
				{{ t('mail', 'Disabled') }}
			</label>
			<input
				id="sieve-enabled"
				v-model="sieveConfig.sieveEnabled"
				type="radio"
				name="sieve-active"
				:disabled="loading"
				:value="true">
			<label class="button"
				for="sieve-enabled"
				:class="{primary: sieveConfig.sieveEnabled}">
				{{ t('mail', 'Enabled') }}
			</label>
		</div>
		<template v-if="sieveConfig.sieveEnabled">
			<label for="sieve-host">{{ t('mail', 'Sieve Host') }}</label>
			<input
				id="sieve-host"
				v-model="sieveConfig.sieveHost"
				type="text"
				required>
			<h4>{{ t('mail', 'Sieve Security') }}</h4>
			<div class="flex-row">
				<input
					id="sieve-sec-none"
					v-model="sieveConfig.sieveSslMode"
					type="radio"
					name="sieve-sec"
					value="none">
				<label
					class="button"
					for="sieve-sec-none"
					:class="{primary: sieveConfig.sieveSslMode === 'none'}">{{
						t('mail', 'None')
					}}</label>
				<input
					id="sieve-sec-ssl"
					v-model="sieveConfig.sieveSslMode"
					type="radio"
					name="sieve-sec"
					value="ssl">
				<label
					class="button"
					for="sieve-sec-ssl"
					:class="{primary: sieveConfig.sieveSslMode === 'ssl'}">
					{{ t('mail', 'SSL/TLS') }}
				</label>
				<input
					id="sieve-sec-tls"
					v-model="sieveConfig.sieveSslMode"
					type="radio"
					name="sieve-sec"
					value="tls">
				<label
					class="button"
					for="sieve-sec-tls"
					:class="{primary: sieveConfig.sieveSslMode === 'tls'}">
					{{ t('mail', 'STARTTLS') }}
				</label>
			</div>
			<label for="sieve-port">{{ t('mail', 'Sieve Port') }}</label>
			<input
				id="sieve-port"
				v-model="sieveConfig.sievePort"
				type="text"
				required>
			<h4>{{ t('mail', 'Sieve Credentials') }}</h4>
			<div class="flex-row">
				<input
					id="sieve-credentials-imap"
					v-model="useImapCredentials"
					type="radio"
					:value="true">
				<label
					class="button"
					for="sieve-credentials-imap"
					:class="{primary: useImapCredentials}">{{
						t('mail', 'IMAP credentials')
					}}</label>
				<input
					id="sieve-credentials-custom"
					v-model="useImapCredentials"
					:value="false"
					type="radio">
				<label
					class="button"
					for="sieve-credentials-custom"
					:class="{primary: !useImapCredentials}">{{
						t('mail', 'Custom')
					}}</label>
			</div>
			<template v-if="!useImapCredentials">
				<label for="sieve-user">{{ t('mail', 'Sieve User') }}</label>
				<input
					id="sieve-user"
					v-model="sieveConfig.sieveUser"
					type="text"
					required>
				<label for="sieve-password">{{
					t('mail', 'Sieve Password')
				}}</label>
				<input
					id="sieve-password"
					v-model="sieveConfig.sievePassword"
					type="password"
					required>
			</template>
		</template>
		<slot name="feedback" />
		<p v-if="errorMessage">
			{{ t('mail', 'Oh Snap!') }}
			{{ errorMessage }}
		</p>
		<input type="submit"
			class="primary"
			:disabled="loading"
			:value="submitButtonText"
			@click.prevent="onSubmit">
	</form>
</template>

<script>
import logger from '../logger'

export default {
	name: 'SieveAccountForm',
	props: {
		account: {
			type: Object,
			required: true,
		},
	},
	data() {
		return {
			sieveConfig: {
				sieveEnabled: this.account.sieveEnabled,
				sieveHost: this.account.sieveHost || this.account.imapHost,
				sievePort: this.account.sievePort || 4190,
				sieveUser: this.account.sieveUser || '',
				sievePassword: '',
				sieveSslMode: this.account.sieveSslMode || 'tls',
			},
			loading: false,
			useImapCredentials: !this.account.sieveUser,
			errorMessage: '',
			submitButtonText: t('mail', 'Save sieve settings'),
		}
	},
	methods: {
		async onSubmit() {
			this.loading = true
			this.errorMessage = ''

			// empty user and password => use imap credentials
			if (this.sieveConfig.sieveUser === '' && this.sieveConfig.sievePassword === '') {
				this.useImapCredentials = true
			}

			// clear user and password if imap credentials are used
			if (this.useImapCredentials) {
				this.sieveConfig.sieveUser = ''
				this.sieveConfig.sievePassword = ''
			}

			try {
				await this.$store.dispatch('updateSieveAccount', { account: this.account, sieveConfig: this.sieveConfig })
			} catch (error) {
				this.errorMessage = error.message
			}

			this.loading = false
		},
	},
}
</script>

<style scoped>
form {
	width: 250px
}
label {
	display: inline-block;
}
input {
	width: 100%;
}

.flex-row {
	display: flex;
}
label.button {
	text-align: center;
	flex-grow: 1;
}

label.error {
	color: red;
}

input[type='radio'] {
	display: none;
}
</style>
