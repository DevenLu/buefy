<template>
    <div class="dropdown" :class="rootClasses">
        <div
            v-if="!inline"
            role="button"
            ref="trigger"
            class="dropdown-trigger"
            @click="toggle"
            @mouseenter="checkHoverable"
            aria-haspopup="true">
            <slot name="trigger"/>
        </div>

        <transition :name="animation">
            <div
                v-if="isMobileModal"
                v-show="isActive"
                class="background"
                :aria-hidden="!isActive"
            />
        </transition>
        <transition :name="animation">
            <div
                v-show="(!disabled && (isActive || isHoverable)) || inline"
                ref="dropdownMenu"
                class="dropdown-menu"
                :aria-hidden="!isActive"
                v-trap-focus="trapFocus">
                <div
                    class="dropdown-content"
                    :role="ariaRoleMenu">
                    <slot/>
                </div>
            </div>
        </transition>
    </div>
</template>

<script>
import trapFocus from '../../directives/trapFocus'
import config from '../../utils/config'

const DEFAULT_CLOSE_OPTIONS = ['escape', 'outside']

export default {
    name: 'BDropdown',
    directives: {
        trapFocus
    },
    props: {
        value: {
            type: [String, Number, Boolean, Object, Array, Function],
            default: null
        },
        disabled: Boolean,
        hoverable: Boolean,
        inline: Boolean,
        position: {
            type: String,
            validator(value) {
                return [
                    'is-top-right',
                    'is-top-left',
                    'is-bottom-left'
                ].indexOf(value) > -1
            }
        },
        mobileModal: {
            type: Boolean,
            default: () => {
                return config.defaultDropdownMobileModal
            }
        },
        ariaRole: {
            type: String,
            default: ''
        },
        animation: {
            type: String,
            default: 'fade'
        },
        multiple: Boolean,
        trapFocus: {
            type: Boolean,
            default: config.defaultTrapFocus
        },
        closeOnClick: {
            type: Boolean,
            default: true
        },
        canClose: {
            type: [Array, Boolean],
            default: true
        },
        expanded: Boolean
    },
    data() {
        return {
            selected: this.value,
            isActive: false,
            isHoverable: this.hoverable,
            _isDropdown: true // Used internally by DropdownItem
        }
    },
    computed: {
        rootClasses() {
            return [this.position, {
                'is-disabled': this.disabled,
                'is-hoverable': this.hoverable,
                'is-inline': this.inline,
                'is-active': this.isActive || this.inline,
                'is-mobile-modal': this.isMobileModal,
                'is-expanded': this.expanded
            }]
        },
        isMobileModal() {
            return this.mobileModal && !this.inline && !this.hoverable
        },
        cancelOptions() {
            return typeof this.canClose === 'boolean'
                ? this.canClose
                    ? DEFAULT_CLOSE_OPTIONS
                    : []
                : this.canClose
        },
        ariaRoleMenu() {
            return this.ariaRole === 'menu' || this.ariaRole === 'list' ? this.ariaRole : null
        }
    },
    watch: {
        /**
        * When v-model is changed set the new selected item.
        */
        value(value) {
            this.selected = value
        },

        /**
        * Emit event when isActive value is changed.
        */
        isActive(value) {
            this.$emit('active-change', value)
        }
    },
    methods: {
        /**
        * Click listener from DropdownItem.
        *   1. Set new selected item.
        *   2. Emit input event to update the user v-model.
        *   3. Close the dropdown.
        */
        selectItem(value) {
            if (this.multiple) {
                if (this.selected) {
                    const index = this.selected.indexOf(value)
                    if (index === -1) {
                        this.selected.push(value)
                    } else {
                        this.selected.splice(index, 1)
                    }
                } else {
                    this.selected = [value]
                }
                this.$emit('change', this.selected)
            } else {
                if (this.selected !== value) {
                    this.selected = value
                    this.$emit('change', this.selected)
                }
            }
            this.$emit('input', this.selected)
            if (!this.multiple) {
                this.isActive = !this.closeOnClick
                if (this.hoverable && this.closeOnClick) {
                    this.isHoverable = false
                }
            }
        },

        /**
        * White-listed items to not close when clicked.
        */
        isInWhiteList(el) {
            if (el === this.$refs.dropdownMenu) return true
            if (el === this.$refs.trigger) return true
            // All chidren from dropdown
            if (this.$refs.dropdownMenu !== undefined) {
                const children = this.$refs.dropdownMenu.querySelectorAll('*')
                for (const child of children) {
                    if (el === child) {
                        return true
                    }
                }
            }
            // All children from trigger
            if (this.$refs.trigger !== undefined) {
                const children = this.$refs.trigger.querySelectorAll('*')
                for (const child of children) {
                    if (el === child) {
                        return true
                    }
                }
            }

            return false
        },

        /**
        * Close dropdown if clicked outside.
        */
        clickedOutside(event) {
            if (this.cancelOptions.indexOf('outside') < 0) return
            if (this.inline) return

            if (!this.isInWhiteList(event.target)) this.isActive = false
        },

        /**
         * Keypress event that is bound to the document
         */
        keyPress(event) {
            // Esc key
            if (this.isActive && event.keyCode === 27) {
                if (this.cancelOptions.indexOf('escape') < 0) return
                this.isActive = false
            }
        },

        /**
        * Toggle dropdown if it's not disabled.
        */
        toggle() {
            if (this.disabled) return

            if (!this.isActive) {
                // if not active, toggle after clickOutside event
                // this fixes toggling programmatic
                this.$nextTick(() => {
                    const value = !this.isActive
                    this.isActive = value
                    // Vue 2.6.x ???
                    setTimeout(() => (this.isActive = value))
                })
            } else {
                this.isActive = !this.isActive
            }
        },

        checkHoverable() {
            if (this.hoverable) {
                this.isHoverable = true
            }
        }
    },
    created() {
        if (typeof window !== 'undefined') {
            document.addEventListener('click', this.clickedOutside)
            document.addEventListener('keyup', this.keyPress)
        }
    },
    beforeDestroy() {
        if (typeof window !== 'undefined') {
            document.removeEventListener('click', this.clickedOutside)
            document.removeEventListener('keyup', this.keyPress)
        }
    }
}
</script>
