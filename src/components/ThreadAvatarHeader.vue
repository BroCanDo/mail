<template>
	<div ref="avatarHeader" class="avatar-header">
		<!-- Participants that can fit in the parent div -->
		<RecipientBubble v-for="participant in participants.slice(0,participantsToDisplay)"
			:key="participant.email"
			:email="participant.email"
			:label="participant.label" />
		<!-- Indicator to show that there are more participants than displayed -->
		<Popover v-if="participants.length > participantsToDisplay" class="avatar-more">
			<span slot="trigger">
				{{ moreParticipantsString }}
			</span>
			<RecipientBubble v-for="participant in participants.slice(participantsToDisplay)"
				:key="participant.email"
				:email="participant.email"
				:label="participant.label" />
		</Popover>
		<!-- Remaining participants, if any (Needed to have avatarHeader reactive) -->
		<RecipientBubble v-for="participant in participants.slice(participantsToDisplay)"
			:key="participant.email"
			class="avatar-hidden"
			:email="participant.email"
			:label="participant.label" />
	</div>
</template>

<script>
import Popover from '@nextcloud/vue/dist/Components/Popover'
import RecipientBubble from './RecipientBubble'
import debounce from 'lodash/fp/debounce'

export default {
	name: 'ThreadAvatarHeader',
	components: {
		RecipientBubble,
		Popover,
	},
	props: {
		participants: {
			type: Array,
			required: true,
		},
	},
	data() {
		return {
			participantsToDisplay: 999,
			resizeDebounced: debounce(500, this.updateParticipantsToDisplay),
		}
	},
	computed: {
		moreParticipantsString() {
			// Returns a number showing the number of thread participants that are not shown in the avatar-header
			return `+${this.participants.length - this.participantsToDisplay}`
		},
	},
	created() {
		window.addEventListener('resize', this.resizeDebounced)
	},
	mounted() {
		this.updateParticipantsToDisplay()
	},
	beforeDestroy() {
		window.removeEventListener('resize', this.resizeDebounced)
	},
	watch: {
		participants() {
			this.updateParticipantsToDisplay()
		}
	},
	methods: {
		updateParticipantsToDisplay() {
			// Wait until everything is in place
			if (!this.$refs.avatarHeader || !this.participants) {
				return
			}

			// Compute the number of participants to display depending on the width available
			const avatarHeader = this.$refs.avatarHeader
			const maxWidth = (avatarHeader.clientWidth - 100) // Reserve 100px for the avatar-more span
			let childrenWidth = 0
			let fits = 0
			let idx = 0
			while (childrenWidth < maxWidth && fits < this.participants.length) {
				// Skipping the 'avatar-more' span
				if (avatarHeader.childNodes[idx].clientWidth === undefined) {
					idx += 3
					continue
				}
				childrenWidth += avatarHeader.childNodes[idx].clientWidth
				fits++
				idx++
			}

			if (childrenWidth > maxWidth) {
				// There's not enough space to show all thread participants
				if (fits > 1) {
					this.participantsToDisplay = fits - 1
				} else if (fits === 0) {
					this.participantsToDisplay = 1
				} else {
					this.participantsToDisplay = fits
				}
			} else {
				// There's enough space to show all thread participants
				this.participantsToDisplay = this.participants.length
			}
		},
	},
}
</script>

<style lang="scss" scoped>
.avatar-header {
	max-height: 24px;
	overflow: hidden;
}

.avatar-more {
	display: inline-block;
	vertical-align: middle;

	span {
		display: inline-block;
		vertical-align: top;
		height: 20px;
		line-height: 20px;
		background-color: var(--color-background-dark);
		border-radius: 10px;
		cursor: pointer;
		padding: 0 4px;
	}
}

.avatar-hidden {
	visibility: hidden;
}

.popover__wrapper {
	max-width: 500px;
}

::v-deep .user-bubble__wrapper:not(:last-child) {
	margin-right: 4px;
}

::v-deep .user-bubble__title {
	cursor: pointer;
}
</style>
