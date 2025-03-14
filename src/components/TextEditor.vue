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
	<ckeditor
		v-if="ready"
		v-model="text"
		:config="config"
		:editor="editor"
		@input="onInput"
		@ready="onEditorReady" />
</template>

<script>
import CKEditor from '@ckeditor/ckeditor5-vue2'
import AlignmentPlugin from '@ckeditor/ckeditor5-alignment/src/alignment'
import Editor from '@ckeditor/ckeditor5-editor-balloon/src/ballooneditor'
import EssentialsPlugin from '@ckeditor/ckeditor5-essentials/src/essentials'
import BlockQuotePlugin from '@ckeditor/ckeditor5-block-quote/src/blockquote'
import BoldPlugin from '@ckeditor/ckeditor5-basic-styles/src/bold'
import HeadingPlugin from '@ckeditor/ckeditor5-heading/src/heading'
import ItalicPlugin from '@ckeditor/ckeditor5-basic-styles/src/italic'
import LinkPlugin from '@ckeditor/ckeditor5-link/src/link'
import ListStyle from '@ckeditor/ckeditor5-list/src/liststyle'
import ParagraphPlugin from '@ckeditor/ckeditor5-paragraph/src/paragraph'

import { getLanguage } from '@nextcloud/l10n'
import DOMPurify from 'dompurify'

import logger from '../logger'

export default {
	name: 'TextEditor',
	components: {
		ckeditor: CKEditor.component,
	},
	props: {
		value: {
			type: String,
			required: true,
		},
		html: {
			type: Boolean,
			default: false,
		},
		placeholder: {
			type: String,
			default: '',
		},
		focus: {
			type: Boolean,
			default: false,
		},
		bus: {
			type: Object,
			required: true,
		},
	},
	data() {
		const plugins = [EssentialsPlugin, ParagraphPlugin]
		const toolbar = ['undo', 'redo']

		if (this.html) {
			plugins.push(...[HeadingPlugin, AlignmentPlugin, BoldPlugin, ItalicPlugin, BlockQuotePlugin, LinkPlugin, ListStyle])
			toolbar.unshift(...['heading', 'alignment', 'bold', 'italic', 'bulletedList', 'numberedList', 'blockquote', 'link'])
		}

		return {
			text: '',
			ready: false,
			editor: Editor,
			config: {
				placeholder: this.placeholder,
				plugins,
				toolbar: {
					items: toolbar,
				},
				language: 'en',
			},
		}
	},
	computed: {
		sanitizedValue() {
			return DOMPurify.sanitize(this.value, {
				FORBID_TAGS: ['style'],
			})
		},
	},
	watch: {
		sanitizedValue(newVal) {
			// needed for reset in composer
			this.text = newVal
		},
	},
	beforeMount() {
		this.loadEditorTranslations(getLanguage())
	},
	methods: {
		async loadEditorTranslations(language) {
			if (language === 'en') {
				// The default, nothing to fetch
				return this.showEditor('en')
			}

			try {
				logger.debug(`loading ${language} translations for CKEditor`)
				await import(
					/* webpackMode: "lazy-once" */
					/* webpackPrefetch: true */
					/* webpackPreload: true */
					`@ckeditor/ckeditor5-build-balloon/build/translations/${language}`
				)
				this.showEditor(language)
			} catch (error) {
				logger.error(`could not find CKEditor translations for "${language}"`, { error })
				this.showEditor('en')
			}
		},
		showEditor(language) {
			logger.debug(`using "${language}" as CKEditor language`)
			this.config.language = language

			this.ready = true
		},
		onEditorReady(editor) {
			const schema = editor.model.schema

			logger.debug('CKEditor editor is ready', { editor, schema })

			this.editorInstance = editor

			// Set 0 pixel margin to all <p> elements
			editor.conversion.for('downcast').add((dispatcher) => {
				dispatcher.on(
					'insert:paragraph',
					(evt, data, conversionApi) => {
						const viewWriter = conversionApi.writer
						viewWriter.setStyle('margin', '0', conversionApi.mapper.toViewElement(data.item))
					},
					{
						priority: 'low',
					}
				)
			})

			schema.on(
				'checkChild',
				(evt, args) => {
					const context = args[0]

					if (context.endsWith('blockQuote')) {
						// Prevent next listeners from being called.
						evt.stop()
						// Set the checkChild()'s return value.
						evt.return = true
					}
				},
				{
					priority: 'highest',
				}
			)
			if (this.focus) {
				logger.debug('focusing TextEditor')
				editor.editing.view.focus()
			}

			// Set value as late as possible, so the custom schema listener is used
			// for the initial editor model
			this.text = this.sanitizedValue

			logger.debug(`setting TextEditor contents to <${this.text}>`)

			this.bus.$on('appendToBodyAtCursor', this.appendToBodyAtCursor)
		},
		onInput() {
			logger.debug(`TextEditor input changed to <${this.text}>`)
			this.$emit('input', this.text)
		},
		appendToBodyAtCursor(toAppend) {
			// https://ckeditor.com/docs/ckeditor5/latest/builds/guides/faq.html#where-are-the-editorinserthtml-and-editorinserttext-methods-how-to-insert-some-content
			const viewFragment = this.editorInstance.data.processor.toView(toAppend)
			const modelFragment = this.editorInstance.data.toModel(viewFragment)
			this.editorInstance.model.insertContent(modelFragment)
		},
	},
}
</script>

<style lang="scss" scoped>
::v-deep a {
	color: var(--ck-color-link-default);
}
::v-deep p {
	cursor: text;
}
</style>
