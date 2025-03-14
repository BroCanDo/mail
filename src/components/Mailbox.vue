<!--
  - @copyright 2020 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @author 2020 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -->

<template>
	<Error v-if="error"
		:error="t('mail', 'Could not open mailbox')"
		message=""
		role="alert" />
	<Loading v-else-if="loadingEnvelopes" :hint="t('mail', 'Loading messages')" role="alert" />
	<Loading
		v-else-if="loadingCacheInitialization"
		:hint="t('mail', 'Loading messages')"
		:slow-hint="t('mail', 'Indexing your messages. This can take a bit longer for larger mailboxes.')" />
	<EmptyMailboxSection v-else-if="isPriorityInbox && !hasMessages" key="empty" />
	<EmptyMailbox v-else-if="!hasMessages" key="empty" />
	<EnvelopeList
		v-else
		:account="account"
		:mailbox="mailbox"
		:search-query="searchQuery"
		:envelopes="envelopesToShow"
		:refreshing="refreshing"
		:loading-more="loadingMore"
		:load-more-button="showLoadMore"
		@delete="onDelete"
		@loadMore="loadMore" />
</template>

<script>
import EmptyMailbox from './EmptyMailbox'
import EnvelopeList from './EnvelopeList'
import Error from './Error'
import { findIndex, propEq } from 'ramda'
import isMobile from '@nextcloud/vue/dist/Mixins/isMobile'
import Loading from './Loading'
import logger from '../logger'
import MailboxLockedError from '../errors/MailboxLockedError'
import MailboxNotCachedError from '../errors/MailboxNotCachedError'
import { matchError } from '../errors/match'
import { wait } from '../util/wait'
import EmptyMailboxSection from './EmptyMailboxSection'
import { showError } from '@nextcloud/dialogs'
import NoTrashMailboxConfiguredError
	from '../errors/NoTrashMailboxConfiguredError'

export default {
	name: 'Mailbox',
	components: {
		EmptyMailboxSection,
		EmptyMailbox,
		EnvelopeList,
		Error,
		Loading,
	},
	mixins: [isMobile],
	props: {
		account: {
			type: Object,
			required: true,
		},
		mailbox: {
			type: Object,
			required: true,
		},
		bus: {
			type: Object,
			required: true,
		},
		paginate: {
			type: String,
			default: 'scroll',
		},
		initialPageSize: {
			type: Number,
			default: 20,
		},
		openFirst: {
			type: Boolean,
			default: true,
		},
		searchQuery: {
			type: String,
			required: false,
			default: undefined,
		},
		isPriorityInbox: {
			type: Boolean,
			required: false,
			default: false,
		},
	},
	data() {
		return {
			error: false,
			refreshing: false,
			loadingMore: false,
			loadingEnvelopes: true,
			loadingCacheInitialization: false,
			loadMailboxInterval: undefined,
			expanded: false,
			endReached: false,
		}
	},
	computed: {
		envelopes() {
			return this.$store.getters.getEnvelopes(this.mailbox.databaseId, this.searchQuery)
		},
		envelopesToShow() {
			if (this.paginate === 'manual' && !this.expanded) {
				return this.envelopes.slice(0, 5)
			}
			return this.envelopes
		},
		hasMessages() {
			return this.envelopes.length > 0
		},
		showLoadMore() {
			return !this.endReached && this.paginate === 'manual'
		},
	},
	watch: {
		account() {
			this.loadEnvelopes()
		},
		mailbox() {
			this.loadEnvelopes()
				.then(() => {
					this.sync(false)
				})
		},
		searchQuery() {
			this.loadEnvelopes()
		},
	},
	created() {
		this.bus.$on('loadMore', this.onScroll)
		this.bus.$on('delete', this.onDelete)
		this.bus.$on('shortcut', this.handleShortcut)
		this.loadMailboxInterval = setInterval(this.loadMailbox, 60000)
	},
	async mounted() {
		this.loadEnvelopes()
			.then(() => {
				this.sync(false)
			})
	},
	destroyed() {
		this.bus.$off('loadMore', this.onScroll)
		this.bus.$off('delete', this.onDelete)
		this.bus.$off('shortcut', this.handleShortcut)
		this.stopInterval()
	},
	methods: {
		initializeCache() {
			this.loadingCacheInitialization = true
			this.error = false

			this.sync(true)
				.then(() => {
					this.loadingCacheInitialization = false

					return this.loadEnvelopes()
				})
		},
		async loadEnvelopes() {
			logger.debug(`fetching envelopes for mailbox ${this.mailbox.databaseId} and query ${this.searchQuery}`, this.mailbox)
			this.loadingEnvelopes = true
			this.loadingCacheInitialization = false
			this.error = false

			try {
				const envelopes = await this.$store.dispatch('fetchEnvelopes', {
					mailboxId: this.mailbox.databaseId,
					query: this.searchQuery,
					limit: this.initialPageSize,
				})

				logger.debug(envelopes.length + ' envelopes fetched', { envelopes })

				this.loadingEnvelopes = false

				if (this.openFirst && !this.isMobile && this.$route.name !== 'message' && envelopes.length > 0) {
					// Show first message
					const first = envelopes[0]

					// Keep the selected account-mailbox combination, but navigate to the message
					// (it's not a bug that we don't use first.accountId and first.mailboxId here)
					logger.debug('showing the first message of mailbox ' + this.$route.params.mailboxId)
					if (first.flags.draft) {
						this.$router.replace({
							name: 'message',
							params: {
								mailboxId: this.$route.params.mailboxId,
								filter: this.$route.params.filter ? this.$route.params.filter : undefined,
								threadId: 'new',
								draftId: first.databaseId,
							},
						})
					} else {
						this.$router.replace({
							name: 'message',
							params: {
								mailboxId: this.$route.params.mailboxId,
								filter: this.$route.params.filter ? this.$route.params.filter : undefined,
								threadId: first.databaseId,
							},
						})
					}
				}
			} catch (error) {
				await matchError(error, {
					[MailboxLockedError.getName()]: async(error) => {
						logger.info('Mailbox is locked', { error })

						await wait(15 * 1000)
						// Keep trying
						await this.loadEnvelopes()
					},
					[MailboxNotCachedError.getName()]: async(error) => {
						logger.info('Mailbox not cached. Triggering initialization', { error })
						this.loadingEnvelopes = false

						try {
							await this.initializeCache()
						} catch (error) {
							logger.error('Could not initialize cache', { error })
							this.error = error
						}
					},
					default: (error) => {
						logger.error('Could not fetch envelopes', { error })
						this.loadingEnvelopes = false
						this.error = error
					},
				})
			}
		},
		async loadMore() {
			if (!this.expanded) {
				logger.debug('expanding envelope list')
				this.expanded = true
				return
			}

			logger.debug('fetching next envelope page')
			this.loadingMore = true

			try {
				const envelopes = await this.$store.dispatch('fetchNextEnvelopePage', {
					mailboxId: this.mailbox.databaseId,
					envelopes: this.envelopes,
					query: this.searchQuery,
				})
				if (envelopes.length === 0) {
					logger.info('envelope list end reached')
					this.endReached = true
				}
			} catch (error) {
				logger.error('could not fetch next envelope page', { error })
			} finally {
				this.loadingMore = false
			}
		},
		async handleShortcut(e) {
			const envelopes = this.envelopes
			const currentId = parseInt(this.$route.params.threadId, 10)

			const env = envelopes.find((e) => e.databaseId === currentId)
			const idx = envelopes.indexOf(env)
			let next

			if (e.srcKey !== 'refresh' && !env) {
				logger.debug('envelope is not in the list, ignoring shortcut', {
					srcKey: e.srcKey,
				})
			}

			switch (e.srcKey) {
			case 'next':
			case 'prev':
				if (e.srcKey === 'next') {
					next = envelopes[idx + 1]
				} else {
					next = envelopes[idx - 1]
				}

				if (!next) {
					logger.debug('ignoring shortcut: head or tail of envelope list reached', {
						envelopes,
						idx,
						srcKey: e.srcKey,
					})
					return
				}

				// Keep the selected account-mailbox combination, but navigate to a different message
				// (it's not a bug that we don't use next.accountId and next.mailboxId here)
				this.$router.push({
					name: 'message',
					params: {
						mailboxId: this.$route.params.mailboxId,
						filter: this.$route.params.filter ? this.$route.params.filter : undefined,
						threadId: next.databaseId,
					},
				})
				break
			case 'del':
				logger.debug('deleting', { env })
				this.onDelete(env.databaseId)
				try {
					await this.$store.dispatch('deleteMessage', {
						id: env.databaseId,
					})
				} catch (error) {
					logger.error('could not delete envelope', {
						env,
						error,
					})

					showError(await matchError(error, {
						[NoTrashMailboxConfiguredError.getName()]() {
							return t('mail', 'No trash mailbox configured')
						},
						default() {
							return t('mail', 'Could not delete message')
						},
					}))
				}

				break
			case 'flag':
				logger.debug('flagging envelope via shortkey', { env })
				this.$store.dispatch('toggleEnvelopeFlagged', env).catch((error) =>
					logger.error('could not flag envelope via shortkey', {
						env,
						error,
					})
				)
				break
			case 'refresh':
				logger.debug('syncing envelopes via shortkey')
				this.sync(false)

				break
			case 'unseen':
				logger.debug('marking message as seen/unseen via shortkey', { env })
				this.$store.dispatch('toggleEnvelopeSeen', { envelope: env }).catch((error) =>
					logger.error('could not mark envelope as seen/unseen via shortkey', {
						env,
						error,
					})
				)
				break
			default:
				logger.warn('shortcut ' + e.srcKey + ' is unknown. ignoring.')
			}
		},
		async sync(init = false) {
			logger.debug('syncing mailbox')
			if (this.refreshing) {
				logger.debug("already sync'ing, aborting")
				return
			}

			this.refreshing = true
			try {
				await this.$store.dispatch('syncEnvelopes', {
					mailboxId: this.mailbox.databaseId,
					query: this.searchQuery,
					init,
				})
			} catch (error) {
				matchError(error, {
					[MailboxLockedError.getName()](error) {
						logger.info('Background sync failed because the mailbox is locked', { error })
					},
					default(error) {
						logger.error('Could not sync envelopes: ' + error.message, { error })
					},
				})
			} finally {
				this.refreshing = false
				logger.debug("finished sync'ing mailbox")
			}
		},
		onDelete(id) {
			const idx = findIndex(propEq('databaseId', id), this.envelopes)
			if (idx === -1) {
				logger.debug('envelope to delete does not exist in envelope list')
				return
			}
			this.envelopes.splice(idx, 1)
			if (id !== this.$route.params.threadId) {
				logger.debug('other message open, not jumping to the next/previous message')
				return
			}

			const next = this.envelopes[idx === 0 ? idx : idx - 1]
			if (!next) {
				logger.debug('no next/previous envelope, not navigating')
				return
			}

			// Keep the selected mailbox, but navigate to a different message
			// (it's not a bug that we don't use next.mailboxId here)
			this.$router.push({
				name: 'message',
				params: {
					mailboxId: this.$route.params.mailboxId,
					filter: this.$route.params.filter ? this.$route.params.filter : undefined,
					threadId: next.databaseId,
				},
			})
		},
		onScroll() {
			if (this.paginate !== 'scroll') {
				logger.debug('ignoring scroll pagination')
				return
			}

			this.loadMore()
		},
		async loadMailbox() {
			// When the account is unified or inbox, return nothing, else sync the mailbox
			if (this.account.isUnified || this.mailbox.specialRole === 'inbox') {
				return
			}
			try {
				await this.sync(false)
			} catch (error) {
				logger.error('Background sync failed: ' + error.message, { error })
			}
		},
		stopInterval() {
			clearInterval(this.loadMailboxInterval)
			this.loadMailboxInterval = undefined
		},
	},
}
</script>

<style lang="scss" scoped>
// Fix vertical space between sections in priority inbox
.nameimportant,
.namestarred {
	::v-deep #load-more-mail-messages {
		margin-top: 0;
		margin-bottom: 8px;
	}
}
</style>
