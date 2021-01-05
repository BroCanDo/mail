<template>
	<MailboxPicker :account="account"
		:selected.sync="destMailboxId"
		:loading="moving"
		:label-select="t('mail', 'Move')"
		:label-select-loading="t('mail', 'Moving')"
		@select="onMove"
		@close="onClose" />
</template>

<script>
import logger from '../logger'
import MailboxPicker from './MailboxPicker'

export default {
	name: 'MoveModal',
	components: {
		MailboxPicker,
	},
	props: {
		account: {
			type: Object,
			required: true,
		},
		envelopes: {
			type: Array,
			required: true,
		},
	},
	data() {
		return {
			moving: false,
			destMailboxId: undefined,
		}
	},
	methods: {
		onClose() {
			this.$emit('close')
		},
		async onMove() {
			this.moving = true

			try {
				const envelopeIds = this.envelopes
					.filter((envelope) => envelope.mailboxId !== this.destMailboxId)
					.map((envelope) => envelope.databaseId)

				if (envelopeIds.length === 0) {
					return
				}

				// Move messages per batch of 50 messages so as to not overload server or create timeouts
				let start = 0
				let batch = []
				do {
					batch = envelopeIds.slice(start, 50)
					await this.$store.dispatch('moveMessages', {
						ids: batch,
						destMailboxId: this.destMailboxId,
					})
					start += 50
				} while (batch.length === 50)

				await this.$store.dispatch('syncEnvelopes', { mailboxId: this.destMailboxId })
				this.$emit('move')
			} catch (error) {
				logger.error('could not move messages', {
					error,
				})
			} finally {
				this.moving = false
				this.$emit('close')
			}
		},
	},
}
</script>
