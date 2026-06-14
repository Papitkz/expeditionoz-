<script setup lang="ts">
  import { ref, computed, onMounted, onUnmounted } from 'vue'
  import { useRouter } from 'vue-router'
  import { useComponentCMS } from '@/composables/useComponentCMS'
  import NoImagePlaceholder from '@/components/NoImagePlaceholder.vue'

  const cms = useComponentCMS('HeroSection')
  const router = useRouter()

  const videos = computed(() =>
    cms.getSection('heroVideos').map((item) => ({
      videoUrl: item.imageUrl || '',
      alt: item.alt || '',
      hasVideo: !!item.imageUrl,
    }))
  )

  // Carousel state
  const currentVideoIndex = ref(0)
  const isTransitioning = ref(false)
  const videoLoaded = ref(false)
  const isPlaying = ref(false)
  const showControls = ref(true)
  const isHovering = ref(false)
  const isMobile = ref(false)
  const touchStartX = ref(0)
  const touchEndX = ref(0)
  const videoError = ref(false)
  const isBuffering = ref(false)
  const videoRef = ref<<HTMLVideoElement | null>(null)
  const showCenterPlay = ref(false) // Hidden by default on mobile
  const showMobilePlay = ref(false) // Controls mobile play button visibility

  let resizeObserver: ResizeObserver | null = null
  let transitionTimeout: number | null = null
  let mobilePlayTimeout: number | null = null

  onMounted(async () => {
    await cms.load()
    checkMobile()

    resizeObserver = new ResizeObserver(() => {
      checkMobile()
    })
    resizeObserver.observe(document.body)

    // Initialize video but DO NOT autoplay
    if (videoRef.value && videos.value[0]?.hasVideo) {
      videoRef.value.src = videos.value[0].videoUrl
      videoRef.value.load()
    }

    document.addEventListener('visibilitychange', handleVisibilityChange)
  })

  onUnmounted(() => {
    if (transitionTimeout) clearTimeout(transitionTimeout)
    if (resizeObserver) resizeObserver.disconnect()
    if (mobilePlayTimeout) clearTimeout(mobilePlayTimeout)
    document.removeEventListener('visibilitychange', handleVisibilityChange)
  })

  // Mobile detection
  const checkMobile = () => {
    isMobile.value = window.innerWidth < 768
    // On desktop, always show play button. On mobile, hide until tapped.
    if (!isMobile.value) {
      showCenterPlay.value = true
    }
  }

  // Computed video properties
  const currentVideoUrl = computed(() => videos.value[currentVideoIndex.value]?.videoUrl || '')
  const nextVideoIndex = computed(() => (currentVideoIndex.value + 1) % videos.value.length)
  const prevVideoIndex = computed(() => currentVideoIndex.value === 0 ? videos.value.length - 1 : currentVideoIndex.value - 1)
  const hasMultipleVideos = computed(() => videos.value.length > 1)

  // Preload next carousel video
  const preloadVideo = (url: string) => {
    const video = document.createElement('video')
    video.src = url
    video.preload = 'metadata'
    video.muted = true
    video.load()
  }

  // Play video
  const playVideo = async () => {
    if (!videoRef.value) return

    try {
      isBuffering.value = true
      videoRef.value.muted = false
      await videoRef.value.play()
      isPlaying.value = true
      videoError.value = false
      showCenterPlay.value = false
      showMobilePlay.value = false
    } catch (err) {
      console.warn('Playback failed:', err)
      isPlaying.value = false
      if (!isMobile.value) showCenterPlay.value = true
    } finally {
      isBuffering.value = false
    }
  }

  // Pause video
  const pauseVideo = () => {
    if (!videoRef.value) return
    videoRef.value.pause()
    isPlaying.value = false
    if (!isMobile.value) showCenterPlay.value = true
  }

  // Toggle play/pause
  const togglePlayPause = () => {
    if (isPlaying.value) {
      pauseVideo()
    } else {
      playVideo()
    }
  }

  // Handle hero section tap on mobile
  const handleHeroTap = () => {
    if (!isMobile.value) return
    // Show play button on tap
    showMobilePlay.value = true
    // Auto-hide after 3 seconds if not clicked
    if (mobilePlayTimeout) clearTimeout(mobilePlayTimeout)
    mobilePlayTimeout = window.setTimeout(() => {
      if (!isPlaying.value) {
        showMobilePlay.value = false
      }
    }, 3000)
  }

  // Smooth video transition
  const switchVideo = (newIndex: number) => {
    if (isTransitioning.value || newIndex === currentVideoIndex.value || !videoRef.value || !videos.value[newIndex]?.hasVideo) return

    isTransitioning.value = true
    isBuffering.value = true

    videoRef.value.pause()

    videoRef.value.style.opacity = '0.3'

    transitionTimeout = window.setTimeout(() => {
      currentVideoIndex.value = newIndex
      videoRef.value!.src = videos.value[newIndex].videoUrl
      videoRef.value!.load()

      const onCanPlay = () => {
        videoRef.value!.removeEventListener('canplaythrough', onCanPlay)
        videoRef.value!.style.opacity = '1'
        isTransitioning.value = false
        isBuffering.value = false

        // Reset to paused state
        isPlaying.value = false
        if (!isMobile.value) showCenterPlay.value = true
        showMobilePlay.value = false

        if (videos.value[nextVideoIndex.value]?.hasVideo) {
          preloadVideo(videos.value[nextVideoIndex.value].videoUrl)
        }
      }

      videoRef.value!.addEventListener('canplaythrough', onCanPlay)
      setTimeout(() => {
        if (isTransitioning.value) {
          onCanPlay()
        }
      }, 3000)

    }, 300)
  }

  const nextVideo = () => switchVideo(nextVideoIndex.value)
  const prevVideo = () => switchVideo(prevVideoIndex.value)
  const goToVideo = (index: number) => switchVideo(index)

  const handleVideoEnded = () => {
    isPlaying.value = false
    if (!isMobile.value) showCenterPlay.value = true
  }

  // Touch handlers for mobile swipe
  const onTouchStart = (e: TouchEvent) => {
    touchStartX.value = e.changedTouches[0].screenX
  }

  const onTouchEnd = (e: TouchEvent) => {
    touchEndX.value = e.changedTouches[0].screenX
    handleSwipe()
  }

  const handleSwipe = () => {
    const swipeThreshold = 50
    const diff = touchStartX.value - touchEndX.value

    if (Math.abs(diff) > swipeThreshold) {
      if (diff > 0) {
        nextVideo()
      } else {
        prevVideo()
      }
    }
  }

  // Hover handlers
  const onMouseEnter = () => {
    isHovering.value = true
    showControls.value = true
  }

  const onMouseLeave = () => {
    isHovering.value = false
  }

  const onVideoLoaded = () => {
    videoLoaded.value = true
    if (videoRef.value) {
      videoRef.value.style.opacity = '1'
    }
  }

  const onVideoError = () => {
    videoError.value = true
    setTimeout(() => {
      if (videoError.value && hasMultipleVideos.value) nextVideo()
    }, 2000)
  }

  const handleVisibilityChange = () => {
    if (document.hidden && isPlaying.value) {
      pauseVideo()
    }
  }
</script>

<<template>
  <section 
    class="relative h-screen w-full overflow-hidden hero-section"
    @mouseenter="onMouseEnter"
    @mouseleave="onMouseLeave"
    @touchstart="onTouchStart"
    @touchend="onTouchEnd"
    @click="handleHeroTap"
  >
    <!-- Video Background with Carousel -->
    <div class="absolute inset-0 z-0">
      <div class="video-container">
        <video
          v-if="videos[0]?.hasVideo"
          ref="videoRef"
          :muted="!isPlaying"
          :loop="false"
          playsinline
          preload="metadata"
          class="w-full h-full object-cover video-hero"
          :class="{ 
            'video-fade-in': videoLoaded,
            'is-buffering': isBuffering
          }"
          @loadeddata="onVideoLoaded"
          @ended="handleVideoEnded"
          @error="onVideoError"
          @waiting="isBuffering = true"
          @playing="isBuffering = false"
          @pause="isPlaying = false; showCenterPlay = true"
        >
          <source :src="currentVideoUrl" type="video/mp4">
        </video>

        <!-- Fallback -->
        <div v-else class="w-full h-full">
          <NoImagePlaceholder label="No Hero Video" class="w-full h-full" />
        </div>

        <!-- Buffering Indicator -->
        <div v-if="isBuffering" class="buffering-indicator">
          <div class="buffering-spinner"></div>
        </div>
      </div>

      <!-- Overlays -->
      <div class="absolute inset-0 bg-[#0A2E4A]/60" />
      <div class="absolute inset-0 bg-gradient-to-b from-black/40 via-transparent to-black/80" />
    </div>

    <!-- DESKTOP CENTER PLAY BUTTON (always visible) -->
    <button
      v-if="videos[0]?.hasVideo && !isBuffering && !isMobile"
      @click.stop="togglePlayPause"
      class="center-play-button desktop-play"
      :aria-label="isPlaying ? 'Pause video' : 'Play video'"
    >
      <div class="play-circle">
        <svg v-if="!isPlaying" width="32" height="32" viewBox="0 0 24 24" fill="currentColor" class="play-icon">
          <path d="M8 5v14l11-7z"/>
        </svg>
        <svg v-else width="28" height="28" viewBox="0 0 24 24" fill="currentColor" class="pause-icon">
          <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
        </svg>
      </div>
      <span v-if="!isPlaying" class="play-label">Play Video</span>
    </button>

    <!-- MOBILE CENTER PLAY BUTTON (hidden until tapped) -->
    <button
      v-if="videos[0]?.hasVideo && !isBuffering && isMobile && showMobilePlay"
      @click.stop="togglePlayPause"
      class="center-play-button mobile-play"
      :class="{ 'is-visible': showMobilePlay }"
      :aria-label="isPlaying ? 'Pause video' : 'Play video'"
    >
      <div class="play-circle">
        <svg v-if="!isPlaying" width="28" height="28" viewBox="0 0 24 24" fill="currentColor" class="play-icon">
          <path d="M8 5v14l11-7z"/>
        </svg>
        <svg v-else width="24" height="24" viewBox="0 0 24 24" fill="currentColor" class="pause-icon">
          <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
        </svg>
      </div>
      <span v-if="!isPlaying" class="play-label">Tap to Play</span>
    </button>

    <!-- Carousel Navigation Arrows (when multiple videos) -->
    <template v-if="hasMultipleVideos && videos[0]?.hasVideo">
      <button
        @click.stop="prevVideo"
        class="carousel-nav carousel-prev"
        aria-label="Previous video"
      >
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="15 18 9 12 15 6" />
        </svg>
      </button>
      <button
        @click.stop="nextVideo"
        class="carousel-nav carousel-next"
        aria-label="Next video"
      >
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="9 18 15 12 9 6" />
        </svg>
      </button>

      <!-- Video Dots -->
      <div class="video-dots">
        <button
          v-for="(_, index) in videos"
          :key="index"
          class="video-dot"
          :class="{ active: currentVideoIndex === index }"
          @click.stop="goToVideo(index)"
          :aria-label="`Go to video ${index + 1}`"
        />
      </div>
    </template>

    <!-- Center Content (Headline, Subtext & Buttons) -->
    <div class="relative z-20 h-full flex flex-col justify-end items-start px-6 sm:px-8 pb-24 md:pb-16">
      <div class="max-w-3xl">
        <!-- Headline -->
        <h1 class="font-display text-white text-4xl md:text-6xl font-light leading-tight tracking-wide">
          <span class="block">WAKE UP ON THE</span>
          <span class="block text-white-400 font-medium">WORLD'S GREATEST REEF.</span>
        </h1>
        <p class="text-gold-400 text-sm md:text-base font-heading tracking-widest uppercase leading-relaxed opacity-90 drop-shadow-md mt-4">
          <span class="block">All inclusive small group expeditions</span>
          <span class="block">exploring Ningaloo Reef by sail.</span>
        </p>
      </div>
      <div class="flex flex-col sm:flex-row gap-4 md:gap-6 mt-8">
        <router-link to="/expeditions" class="btn-hero-choose px-8 py-4 text-sm md:text-base tracking-widest font-medium uppercase text-center">
          Choose Your Adventure
        </router-link>
        <button @click.stop="router.push('/contact')" class="btn-hero-book px-8 py-4 text-sm md:text-base tracking-widest font-medium uppercase text-center">
          Booking Enquiry
        </button>
      </div>
    </div>
  </section>
</template>

<style scoped>
:root {
  --color-gold-400: #C9A84C;
  --color-sand-100: #F8F5EF;
  --font-display: 'Playfair Display', serif; 
  --font-heading: 'Montserrat', sans-serif; 
}

.hero-section {
  min-height: 100dvh;
}

.video-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.video-hero {
  transition: opacity 1.5s ease;
  will-change: transform, opacity;
}

.video-fade-in {
  opacity: 1;
  animation: slowZoom 20s ease-out forwards;
}

.video-hero.is-buffering {
  opacity: 0.5;
}

@keyframes slowZoom {
  0% { transform: scale(1); }
  100% { transform: scale(1.08); }
}

/* CENTER PLAY BUTTON - SHARED */
.center-play-button {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 25;
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  transition: opacity 0.4s ease, transform 0.3s ease;
}

/* DESKTOP PLAY BUTTON */
.desktop-play {
  opacity: 1;
  pointer-events: auto;
}

/* MOBILE PLAY BUTTON */
.mobile-play {
  opacity: 0;
  pointer-events: none;
  transform: translate(-50%, -50%) scale(0.9);
}

.mobile-play.is-visible {
  opacity: 1;
  pointer-events: auto;
  transform: translate(-50%, -50%) scale(1);
}

.play-circle {
  width: 80px;
  height: 80px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.12);
  backdrop-filter: blur(10px);
  border: 2px solid rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  box-shadow: 0 4px 30px rgba(0, 0, 0, 0.3);
}

.center-play-button:hover .play-circle {
  background: rgba(255, 255, 255, 0.22);
  border-color: rgba(255, 255, 255, 0.95);
  transform: scale(1.1);
  box-shadow: 0 8px 40px rgba(0, 0, 0, 0.4);
}

.play-icon {
  color: white;
  margin-left: 4px;
}

.pause-icon {
  color: white;
}

.play-label {
  color: rgba(255, 255, 255, 0.9);
  font-size: 0.75rem;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  font-weight: 500;
  text-shadow: 0 2px 8px rgba(0, 0, 0, 0.5);
}

/* CAROUSEL NAVIGATION ARROWS */
.carousel-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 20;
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: rgba(4, 26, 43, 0.5);
  backdrop-filter: blur(6px);
  border: 1px solid rgba(198, 168, 76, 0.4);
  color: #C9A84C;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
}

.carousel-nav:hover {
  background: rgba(198, 168, 76, 0.15);
  border-color: #C9A84C;
  transform: translateY(-50%) scale(1.08);
}

.carousel-prev {
  left: 16px;
}

.carousel-next {
  right: 16px;
}

/* VIDEO DOTS */
.video-dots {
  position: absolute;
  bottom: 140px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 20;
  display: flex;
  gap: 10px;
}

.video-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  border: 1px solid rgba(198, 168, 76, 0.5);
  background: transparent;
  cursor: pointer;
  transition: all 0.3s ease;
  padding: 0;
}

.video-dot.active {
  background: #C9A84C;
  border-color: #C9A84C;
  transform: scale(1.2);
}

.buffering-indicator {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 5;
  pointer-events: none;
}

.buffering-spinner {
  width: 40px;
  height: 40px;
  border: 2px solid rgba(201, 168, 76, 0.3);
  border-top-color: #C9A84C;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.text-gold-400 {
  color: var(--color-gold-400);
}

/* Hero Buttons */
.btn-hero-choose {
  background-color: transparent;
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.6);
  border-radius: 4px;
  transition: all 0.3s ease;
  text-decoration: none;
  display: inline-block;
  backdrop-filter: blur(4px);
}

.btn-hero-choose:hover {
  border-color: white;
  background-color: rgba(255, 255, 255, 0.1);
  transform: translateY(-2px);
}

.btn-hero-book {
  background-color: var(--color-gold-400);
  color: #0A2E4A;
  border: 1px solid var(--color-gold-400);
  border-radius: 4px;
  transition: all 0.3s ease;
  display: inline-block;
  box-shadow: 0 4px 15px rgba(201, 168, 76, 0.3);
}

.btn-hero-book:hover {
  background-color: #dbb86a;
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(201, 168, 76, 0.4);
}

@media (max-width: 767px) {
  .play-circle {
    width: 60px;
    height: 60px;
  }
  .play-icon {
    width: 24px;
    height: 24px;
    margin-left: 2px;
  }
  .pause-icon {
    width: 20px;
    height: 20px;
  }
  .play-label {
    font-size: 0.7rem;
  }
  .carousel-nav {
    width: 36px;
    height: 36px;
  }
  .carousel-prev {
    left: 8px;
  }
  .carousel-next {
    right: 8px;
  }
  .video-dots {
    bottom: 120px;
  }
}
</style>
