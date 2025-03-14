<!--
  - @copyright 2019 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @author 2019 Christoph Wurst <christoph@winzerhof-wurst.at>
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
	<AppNavigation>
		<AppNavigationNew
			:text="t('mail', 'New message')"
			button-id="mail_new_message"
			button-class="icon-add"
			role="complementary"
			@click="onNewMessage" />
		<template #list>
			<ul id="accounts-list">
				<template v-for="group in menu">
					<NavigationAccount
						v-if="group.account"
						:key="group.account.id"
						:account="group.account"
						:first-mailbox="group.mailboxes[0]"
						:is-first="isFirst(group.account)"
						:is-last="isLast(group.account)" />
					<template v-for="item in group.mailboxes">
						<NavigationMailbox
							v-show="
								!group.isCollapsible ||
									!group.account.collapsed ||
									!isCollapsed(group.account, item)
							"
							:key="'mailbox-' + item.databaseId"
							:account="group.account"
							:mailbox="item" />
						<NavigationMailbox
							v-if="!group.account.isUnified && item.specialRole === 'inbox'"
							:key="item.databaseId + '-starred'"
							:account="group.account"
							:mailbox="item"
							filter="starred" />
					</template>
					<NavigationAccountExpandCollapse
						v-if="!group.account.isUnified && group.isCollapsible"
						:key="'collapse-' + group.account.id"
						:account="group.account" />
					<AppNavigationSpacer :key="'spacer-' + group.account.id" />
				</template>
			</ul>
		</template>
		<template #footer>
			<AppNavigationSettings :title="t('mail', 'Settings')">
				<AppSettingsMenu />
			</AppNavigationSettings>
		</template>
	</AppNavigation>
</template>

<script>
import AppNavigation from '@nextcloud/vue/dist/Components/AppNavigation'
import AppNavigationNew from '@nextcloud/vue/dist/Components/AppNavigationNew'
import AppNavigationSettings
	from '@nextcloud/vue/dist/Components/AppNavigationSettings'
import AppNavigationSpacer
	from '@nextcloud/vue/dist/Components/AppNavigationSpacer'

import logger from '../logger'
import NavigationAccount from './NavigationAccount'
import NavigationAccountExpandCollapse from './NavigationAccountExpandCollapse'
import NavigationMailbox from './NavigationMailbox'

import AppSettingsMenu from '../components/AppSettingsMenu'

export default {
	name: 'Navigation',
	components: {
		AppNavigation,
		AppNavigationNew,
		AppNavigationSettings,
		AppNavigationSpacer,
		AppSettingsMenu,
		NavigationAccount,
		NavigationAccountExpandCollapse,
		NavigationMailbox,
	},
	computed: {
		menu() {
			return this.$store.getters.accounts.map((account) => {
				const mailboxes = this.$store.getters.getMailboxes(account.id)
				const nonSpecialRoleMailboxes = mailboxes.filter(
					(mailbox) => this.isCollapsed(account, mailbox)
				)
				const isCollapsible = nonSpecialRoleMailboxes.length > 1

				return {
					id: account.id,
					account,
					mailboxes,
					isCollapsible,
				}
			})
		},
	},
	methods: {
		isCollapsed(account, mailbox) {
			if (mailbox.specialRole === 'inbox') {
				// INBOX is always visible
				return false
			}

			if (mailbox.databaseId === account.draftsMailboxId
				|| mailbox.databaseId === account.sentMailboxId
				|| mailbox.databaseId === account.trashMailboxId) {
				// Special folders are always visible
				return false
			}

			return true
		},
		onNewMessage() {
			const accountId = this.$route.params.accountId || this.$store.getters.accounts[0].id

			const mailboxId = this.$route.params.mailboxId || this.$store.getters.getMailboxes(accountId)[0]?.databaseId
			if (
				this.$router.currentRoute.name === 'message'
				&& this.$router.currentRoute.params.getMailboxes === 'new'
			) {
				// If we already show the composer, navigating to it would be pointless (and doesn't work)
				// instead trigger an event to reset the composer
				this.$root.$emit('newMessage')
				return
			}

			this.$router
				.push({
					name: 'message',
					params: {
						mailboxId,
						filter: this.$route.params.filter ? this.$route.params.filter : undefined,
						threadId: 'new',
					},
				})
				.catch((err) => {
					logger.error(err)
				})
		},
		isFirst(account) {
			const accounts = this.$store.getters.accounts
			return account === accounts[1]
		},
		isLast(account) {
			const accounts = this.$store.getters.accounts
			return account === accounts[accounts.length - 1]
		},
	},
}
</script>

<style scoped>
</style>
