<template>
    <div ref="dropdownElement" class="dropdown" :style="cssVariables" ww-responsive="dropdown" @click.stop>
        <div
            :style="{ display: !isMenuDisplayed ? 'block' : 'none' }"
            class="dropdown-default"
            ww-responsive="dropdown-desktop"
            @click="handleClickInside"
            @mouseenter="handleMouseHover(true)"
            @mouseleave="handleMouseHover(false)"
        >
            <div class="dropdown-hover-trigger">
                <wwLayout class="dropdown__layout" path="dropdown">
                    <template #default="{ item }">
                        <wwLayoutItem>
                            <wwElement v-bind="item" :states="isVisible ? ['On Hover'] : []"></wwElement>
                        </wwLayoutItem>
                    </template>
                </wwLayout>
                <Teleport :to="rootComponent">
                    <transition :name="content.appearAnimation" mode="out-in">
                        <div
                            :style="{ display: isVisible ? 'block' : 'none', ...this.contentPositionStyles() }"
                            ww-responsive="dropdown-content-desktop"
                            class="dropdown__content under"
                        >
                            <wwLayout
                                ref="dropdownContent"
                                class="layout"
                                path="dropdownContent"
                                @mouseenter="isMouseInContent = true"
                                @mouseleave="isMouseInContent = false"
                            >
                                <template #default="{ item }">
                                    <wwLayoutItem>
                                        <wwElement v-bind="item" :states="states"></wwElement>
                                    </wwLayoutItem>
                                </template>
                            </wwLayout>
                        </div>
                    </transition>
                </Teleport>
            </div>
        </div>
        <div
            :style="{ display: isMenuDisplayed ? 'block' : 'none' }"
            class="dropdown-mobile"
            ww-responsive="dropdown-mobile"
            @click="handleMobileClick"
        >
            <wwLayout class="dropdown__layout--mobile" path="dropdown">
                <template #default="{ item }">
                    <wwLayoutItem>
                        <wwElement v-bind="item" :states="states"></wwElement>
                    </wwLayoutItem>
                </template>
            </wwLayout>

            <div class="dropdown__content--mobile">
                <wwExpandTransition>
                    <wwLayout
                        :style="{ display: isVisible ? 'flex' : 'none' }"
                        ww-responsive="dropdown-content-mobile"
                        ref="dropdownContent"
                        class="layout"
                        path="dropdownContent"
                        @mouseenter="isMouseInContent = true"
                        @mouseleave="isMouseInContent = false"
                    >
                        <template #default="{ item }">
                            <wwLayoutItem>
                                <wwElement v-bind="item" :states="states"></wwElement>
                            </wwLayoutItem>
                        </template>
                    </wwLayout>
                </wwExpandTransition>
            </div>
        </div>
    </div>
</template>

<script>
import wwExpandTransition from './wwExpandTransition.vue';
import { Teleport } from 'vue';

export default {
    components: { wwExpandTransition, Teleport },
    emits: ['update:content:effect'],
    props: {
        content: { type: Object, required: true },
        wwFrontState: { type: Object, required: true },
        /* wwEditor:start */
        wwEditorState: { type: Object, required: true },
        /* wwEditor:end */
    },
    data() {
        return {
            isVisible: false,
            topPosition: 0,
            states: [],
            isMouseIn: false,
            isMouseInContent: false,
            isMounted: false,
        };
    },
    computed: {
        rootComponent() {
            return document.getElementById('app') || document.body;
        },

        isEditing() {
            /* wwEditor:start */
            return this.wwEditorState.editMode === wwLib.wwEditorHelper.EDIT_MODES.EDITION;
            /* wwEditor:end */
            // eslint-disable-next-line no-unreachable
            return false;
        },
        isMenuDisplayed() {
            if (!this.content.isMenuBreakpoint) return false;
            if (this.content.menuBreakpoint === 'laptop') return true;
            if (this.content.menuBreakpoint === 'tablet')
                return this.wwFrontState.screenSize === 'mobile' || this.wwFrontState.screenSize === 'tablet';
            return this.wwFrontState.screenSize === 'mobile';
        },
        cssVariables() {
            const perspectiveValue = () => {
                let perspective = 'none';

                if (this.content.appearAnimation === 'rotate') {
                    perspective = '2000px';
                }

                return perspective;
            };

            let leftValue = '50%';
            let transformValue = 'translateX(-50%)';

            if (this.content.alignement === 'align-left') {
                leftValue = '0px';
                transformValue = 'translateX(0%)';
            } else if (this.content.alignement === 'center') {
                leftValue = '50%';
                transformValue = 'translateX(-50%)';
            } else if (this.content.alignement === 'align-right') {
                leftValue = '100%';
                transformValue = 'translateX(-100%)';
            }

            return {
                '--top-position': this.topPosition + 'px',
                '--content-dimension': this.isVisible ? '300px' : '0px',
                '--animation-duration': this.content.animationDuration,
                '--perspective': perspectiveValue(),
                '--animationTimingFunction': this.content.animationTimingFunction,
                '--slideOrigin': this.content.slideOrigin,
                '--rotationAngle': this.content.rotationAngle,
                '--left-value': leftValue,
                '--transform-value': transformValue,
            };
        },
    },
    watch: {
        /* wwEditor:start */
        content: {
            deep: true,
            handler: function () {
                this.updatePosition();
            },
        },
        isEditing() {
            if (!this.isEditing) {
                this.isVisible = false;
            }
            this.updatePosition();
        },
        /* wwEditor:end */
        isVisible(value) {
            // eslint-disable-next-line vue/custom-event-name-casing
            if (value) {
                wwLib.$emit('ww-hover-dropdown:opened', this.id);
                // this.$emit('update:content:effect', { internalDisplay: value });
                this.updatePosition();
            }
        },
        'content.internalDisplay'(value) {
            this.isVisible = value;
        },
        'content.alignement'(value) {
            console.log(value);
        },
        isMouseIn(value) {
            if (this.content.trigger === 'mouseenter') {
                this.isVisible = value;
            }
        },
    },
    setup() {
        const id = wwLib.wwUtils.getUid();
        return { id };
    },
    beforeMount() {
        wwLib.getFrontDocument().addEventListener('click', this.handleClickOutside);
        wwLib.$on('ww-hover-dropdown:opened', dropdownId => {
            if (dropdownId !== this.id) {
                this.isVisible = false;
                this.states = [];
            }
        });
    },
    mounted() {
        this.isVisible = this.content.internalDisplay;
        this.updatePosition();
        this.isMounted = true;
    },
    unmounted() {
        wwLib.$off('ww-hover-dropdown:opened');
        if (this.content.trigger === 'click') {
            wwLib.getFrontDocument().removeEventListener('click', this.handleClickOutside);
        }
    },
    methods: {
        parentOffset() {
            if (!this.isMounted) return null;
            return this.$refs.dropdownElement.getBoundingClientRect();
        },

        contentPositionStyles() {
            const parentOffset = this.parentOffset();

            return {
                position: 'absolute',
                top: parentOffset == null ? 0 : parentOffset.top + this.content.contentOffsetY + 'px',
                left:
                    parentOffset == null
                        ? 0
                        : parentOffset.left + parentOffset.width + this.content.contentOffsetX + 'px',
            };
        },

        handleClickInside() {
            if (this.isVisible && this.isMouseInContent && this.content.closeOnClick === 'outside') return;
            if (this.content.trigger === 'click') {
                this.isVisible = !this.isVisible;
            }
        },
        handleClickOutside() {
            if (this.isMouseIn) return;
            this.isVisible = false;
        },
        handleMouseHover(value) {
            this.isMouseIn = value;
            if (this.content.trigger === 'mouseenter') {
                this.isVisible = value;
            }
        },
        handleMobileClick() {
            if (this.isVisible && this.isMouseInContent && this.content.closeOnClick === 'outside') return;
            this.isVisible = !this.isVisible;
        },
        updatePosition() {
            this.topPosition = this.$refs.dropdownElement.getBoundingClientRect().top;
        },
    },
};
</script>

<style lang="scss" scoped>
:root {
    --animation-duration: 300ms;
    --top-position: 0px;
    --perspective: 2000px;
    --slideOrigin: -20px;
    --rotationAngle: -35deg;
}

.dropdown {
    position: relative;

    .dropdown-default {
        perspective: var(--perspective);
    }

    &__layout {
        display: flex;
        flex-wrap: nowrap;
        flex-direction: row;
        justify-content: center;
        align-items: center;
    }
    &__content {
        z-index: 100;
        position: fixed;
        top: var(--top-position);
        left: var(--left-value);
        transform: var(--transform-value);
        margin-top: -1px;
        flex-direction: row;
        justify-content: center;
        align-content: center;

        .layout {
            display: flex;
            flex-wrap: nowrap;
            flex-direction: row;
            justify-content: center;
            align-items: center;
            min-height: 20px;
            min-width: 100px;
        }
    }
    &__content.under {
        position: absolute;
        top: 100%;

        .layout {
            display: flex;
            flex-direction: column;
            min-height: 20px;
            min-width: 100px;
        }
    }
    .dropdown-mobile {
        width: 100%;
        &__content {
            z-index: 9;
            position: fixed;
            top: var(--top-position);
            left: 0;
            width: 80vw;
            margin-left: 10vw;
            margin-top: -1px;
            display: flex;
            flex-direction: row;
            justify-content: center;
            align-content: center;
            .layout {
                display: flex;
                flex-direction: column;
                min-height: 20px;
                min-width: 100px;
            }
        }
    }
}

/* FADE */
.fade-enter-active,
.fade-leave-active {
    transition: all var(--animation-duration) var(--animationTimingFunction);
}

.fade-enter-from,
.fade-leave-to {
    opacity: 0;
}

/* SLIDE Y */
.slideY-enter-active,
.slideY-leave-active {
    transition: all var(--animation-duration) var(--animationTimingFunction);
}

.slideY-enter-from {
    opacity: 0;
    transform: translateX(-50%) translateY(var(--slideOrigin));
}
.slideY-leave-to {
    opacity: 0;
    transform: translateX(-50%) translateY(var(--slideOrigin));
}

/* SLIDE X */
.slideX-enter-active,
.slideX-leave-active {
    transition: all var(--animation-duration) var(--animationTimingFunction);
}

.slideX-enter-from {
    opacity: 0;
    transform: translateX(calc(-50% + var(--slideOrigin)));
}
.slideX-leave-to {
    opacity: 0;
    transform: translateX(calc(-50% + var(--slideOrigin)));
}

//  SCALE DOWN
.scale-enter-active,
.scale-leave-active {
    transition: all var(--animation-duration) var(--animationTimingFunction);
    animation-fill-mode: forwards;
    transform-origin: top center;
}

.scale-enter-from {
    opacity: 0;
    transform: translateX(-50%) scale(0);
}
.scale-leave-to {
    opacity: 0;
    transform: translateX(-50%) scale(0);
}

//  ROTATE X
.rotate-enter-active,
.rotate-leave-active {
    transition: all var(--animation-duration) var(--animationTimingFunction);
    transform-origin: center -20px;
}

.rotate-enter-from {
    opacity: 0;
    transform: translateX(-50%) rotateX(var(--rotationAngle));
}
.rotate-leave-to {
    opacity: 0;
    transform: translateX(-50%) rotateX(var(--rotationAngle));
}
</style>
