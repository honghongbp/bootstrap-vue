<template>
    <div class="carousel slide"
         role="region"
         :id="id || null"
         :style="{background,height}"
         :aria-busy="isSliding ? 'true' : 'false'"
         @mouseenter="pause"
         @mouseleave="start"
         @focusin="pause"
         @focusout="restart"
         @keydown.left.stop.prevent="prev"
         @keydown.right.stop.prevent="next"
    >

        <!-- Wrapper for slides -->
        <div class="carousel-inner"
             role="list"
             ref="inner"
             :id="id ? (id + '__BV_inner_') : null"
        >
            <slot></slot>
        </div>

        <!-- Controls -->
        <template v-if="controls">
            <a class="carousel-control-prev"
               href="#"
               role="button"
               :aria-controls="id ? (id + '__BV_inner_') : null"
               @click.stop.prevent="prev"
               @keydown.enter.stop.prevent="prev"
               @keydown.space.stop.prevent="prev"
            >
                <span class="carousel-control-prev-icon" aria-hidden="true"></span>
                <span class="sr-only">{{labelPrev}}</span>
            </a>
            <a class="carousel-control-next"
               href="#"
               role="button"
               :aria-controls="id ? (id + '__BV_inner_') : null"
               @click.stop.prevent="next"
               @keydown.enter.stop.prevent="next"
               @keydown.space.stop.prevent="next"
            >
                <span class="carousel-control-next-icon" aria-hidden="true"></span>
                <span class="sr-only">{{labelNext}}</span>
            </a>
        </template>

        <!-- Indicators -->
        <ol class="carousel-indicators"
            role="group"
            v-show="indicators"
            :id="id ? (id + '__BV_indicators_') : null"
            :aria-hidden="indicators ? 'false' : 'true'"
            :aria-label="(indicators && labelIndicators) ? labelIndicators : null"
            :aria-owns="(indicators && id) ? (id + '__BV_inner_') : null"
        >
            <li v-for="n in slides.length"
                :key="'slide_' + n"
                role="button"
                :id="id ? (id + '__BV_indicator_' + n + '_') : null"
                :tabindex="indicators ? '0' : '-1'"
                :class="{active:n-1 === index}"
                :aria-current="n-1 === index ? 'true' : 'false'"
                :aria-label="labelGotoSlide + ' ' + n"
                :aria-describedby="slides[n-1].id || null"
                :aria-controls="id ? (id + '__BV_inner_') : null"
                @click="setSlide(n-1)"
                @keydown.enter.stop.prevent="setSlide(n-1)"
                @keydown.space.stop.prevent="setSlide(n-1)"
            ></li>
        </ol>

    </div>
</template>

<script>
    import { from as arrayFrom } from '../utils/array';
    import { observeDom } from '../utils';

    // Slide directional classes
    const DIRECTION = {
        next: {
            dirClass: 'carousel-item-left',
            overlayClass: 'carousel-item-next'
        },
        prev: {
            dirClass: 'carousel-item-right',
            overlayClass: 'carousel-item-prev'
        }
    };
    
    // Fallback Transition duration (with a little buffer) in ms
    const TRANS_DURATION = 600 + 50;

    // Transition Event names
    const TransitionEndEvents = {
        WebkitTransition: 'webkitTransitionEnd',
        MozTransition: 'transitionend',
        OTransition: 'otransitionend',
        transition: 'transitionend'
    };

    // Return the brtowser specific transitionend event name
    function getTransisionEndEvent(el) {
        for (const name in TransitionEndEvents) {
            if (el.style[name] !== undefined) {
                return TransitionEndEvents[name];
            }
        }
        // fallback
        return null;
    }

    export default {
        data() {
            return {
                index: this.value || 0,
                isSliding: false,
                intervalId: null,
                transitionEndEvent: null,
                slides: []
            };
        },
        props: {
            id: {
                type: String
            },
            labelPrev: {
                type: String,
                default: 'Previous Slide'
            },
            labelNext: {
                type: String,
                default: 'Next Slide'
            },
            labelGotoSlide: {
                type: String,
                default: 'Goto Slide'
            },
            labelIndicators: {
                type: String,
                default: 'Select a slide to display'
            },
            interval: {
                type: Number,
                default: 5000
            },
            indicators: {
                type: Boolean,
                default: false
            },
            controls: {
                type: Boolean,
                default: false
            },
            height: {
                type: String
            },
            background: {
                type: String
            },
            value: {
                type: Number,
                default: 0
            }
        },
        computed: {
            isCycling() {
                return Boolean(this.intervalId);
            }
        },
        methods: {
            // Set slide
            setSlide(slide) {
                // Don't animate when page is not visible
                if (typeof document !== 'undefined' && document.visibilityState && document.hidden) {
                    return;
                }

                const len = this.slides.length;
                
                // Don't do anything if nothing to slide to
                if (len === 0) {
                    return;
                }
                
                // Don't change slide while transitioning, wait until transition is done
                if (this.isSliding) {
                    // Schedule slide after sliding complete
                    this.$once('slid', () => this.setSlide(slide));
                    return;
                }

                // Make sure we have an integer (you never know!)
                slide = Math.floor(slide);

                // Set new slide index. Wrap around if necessary
                this.index = slide >= len ? 0 : (slide >= 0 ? slide : len - 1);
            },

            // Previous slide
            prev() {
                this.setSlide(this.index - 1);
            },

            // Next slide
            next() {
                this.setSlide(this.index + 1);
            },
            
            // Pause auto rotation
            pause() {
                if (this.isCycling) {
                    clearInterval(this.intervalId);
                    this.intervalId = null;

                    // Make current slide focusable for screen readers
                    this.slides[this.index].tabIndex = 0;
                }
            },

            // Start auto rotate slides
            start() {
                // Don't start if no intetrval, or if we are already running
                if (!Boolean(this.interval) || this.isCycling) {
                    return;
                }
                this.slides.forEach(slide => {
                    slide.tabIndex = -1;
                });
                this.intervalId = setInterval(() => {
                    this.next();
                }, Math.min(1000, this.interval));
            },

            // Re-Start auto rotate slides when focus/hover leaves the carousel
            restart(evt) {
                if (!evt.relatedTarget || !this.$el.contains(evt.relatedTarget)) {
                    this.start();
                }
            },

            // Update slide list
            updateSlides() {
                this.pause();

                // Get all slides
                this.slides = arrayFrom(this.$refs.inner.querySelectorAll('.carousel-item'));

                const id = this.id;
                const numSlides = this.slides.length;

                // Keep slide number in range
                const index = Math.max(0, Math.min(Math.floor(this.index), numSlides - 1));

                this.slides.forEach((slide, idx) => {
                    const n = idx + 1;

                    slide.classList[idx === index ? 'add' : 'remove']('active');
                    slide.setAttribute('aria-current', idx === index ? 'true' : 'false');
                    slide.setAttribute('aria-posinset', String(n));
                    slide.setAttribute('aria-setsize', String(numSlides));
                    slide.tabIndex = -1;
                    if (id) {
                        slide.setAttribute('aria-controlledby', id + '__BV_indicator_' + n + '_');
                    }
                });

                // Set slide as active
                this.setSlide(index);

                this.start();
            }

        },
        watch: {
            value(newVal, oldVal) {
                if (newVal !== oldVal) {
                    this.setSlide(newVal);
                }
            },
            interval(newVal, oldVal) {
                if (newVal === oldVal) {
                    return;
                }
                if (!Boolean(newVal)) {
                    // Pausing slide show
                    this.pause();
                } else {
                    // Restarting or Changing interval
                    this.pause();
                    this.start();
                }
            },
            index(val, oldVal) {
                if (val === oldVal || this.isSliding) {
                    return;
                }

                // Determine sliding direction
                let direction = (val > oldVal) ? DIRECTION.next : DIRECTION.prev;

                // Rotates
                if (oldVal === 0 && val === this.slides.length - 1) {
                    direction = DIRECTION.prev;
                } else if (oldVal === this.slides.length - 1 && val === 0) {
                    direction = DIRECTION.next;
                }

                // Determine current and next slides
                const currentSlide = this.slides[oldVal];
                const nextSlide = this.slides[val];

                // Don't do anything if there aren't any slides to slide to
                if (!currentSlide || !nextSlide) {
                    return;
                }

                // Start animating
                this.isSliding = true;
                this.$emit('slide', val);

                // Update v-model
                this.$emit('input', this.index);

                nextSlide.classList.add(direction.overlayClass);
                // Trigger a reflow of next slide
                // eslint-ignore-next-line no-void
                void(nextSlide.offsetHeight);

                currentSlide.classList.add(direction.dirClass);
                nextSlide.classList.add(direction.dirClass);

                // Transition End handler
                let called = false;
                const onceTransEnd = (evt) => {
                    if (called) {
                        return;
                    }
                    called = true;
                    if (this.transitionEndEvent) {
                        currentSlide.removeEventListener(this.transitionEndEvent, onceTransEnd);
                    }
                    this._animationTimeout = null;

                    nextSlide.classList.remove(direction.dirClass);
                    nextSlide.classList.remove(direction.overlayClass);
                    nextSlide.classList.add('active');

                    currentSlide.classList.remove('active');
                    currentSlide.classList.remove(direction.dirClass);
                    currentSlide.classList.remove(direction.overlayClass);

                    currentSlide.setAttribute('aria-current', 'false');
                    nextSlide.setAttribute('aria-current', 'true');
                    currentSlide.setAttribute('aria-hidden', 'true');
                    nextSlide.setAttribute('aria-hidden', 'false');

                    currentSlide.tabIndex = -1;
                    nextSlide.tabIndex = -1;

                    if (!this.isCycling) {
                        // Focus the next slide for screen readers if not in play mode
                        nextSlide.tabIndex = 0;
                        this.$nextTick(() => {
                            nextSlide.focus();
                        });
                    }

                    this.isSliding = false;
                    // Notify ourselves that we're done sliding (slid)
                    this.$nextTick(() => this.$emit('slid', val));
                };

                // Clear transition classes after transition ends
                if (this.transitionEndEvent) {
                    currentSlide.addEventListener(this.transitionEndEvent, onceTransEnd);
                }
                // Fallback to setTimeout
                this._animationTimeout = setTimeout(onceTransEnd, TRANS_DURATION);
            }
        },
        created() {
            // Create private non-reactive props
            this._animationTimeout = null;
        },
        mounted() {
            // Cache current browser transitionend event name
            this.transitionEndEvent = getTransisionEndEvent(this.$el) || null;

            // Get all slides
            this.updateSlides();

            // Observe child changes so we can update slide list
            observeDom(this.$refs.inner, this.updateSlides.bind(this), {subtree: false});
        },
        destroyed() {
            clearInterval(this.intervalId);
            clearTimeout(this._animationTimeout);
            this._animationTimeout = null;
        }
    };

</script>
