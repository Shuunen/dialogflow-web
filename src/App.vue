<template>
<section id="app" :class="{ closeable: Config.features.closeable }">

    <div class="app-wrapper" v-show="displayed">
            
        <div class="app-title" v-if="Config.features.title">
            {{ $t("title") }}
        </div>

        <!-- The input -->
        <div class="query" :class="{ bottom: Config.features.inputBottom }">
            <div class="wrapper" v-if="micro == false">
                <i class="material-icons iicon" v-if="Config.features.recognition" @click="microphone(true)">mic</i>
                <input :aria-label="$t('generic.inputPlaceholder')" autocomplete="off" v-model="query" class="queryform" @keyup.enter="submit()" :placeholder="$t('generic.inputPlaceholder')" autofocus type="text">
                <i class="material-icons iicon t2s" @click="mute(true)" v-if="Config.features.speech && muted == false">volume_up</i>
                <i class="material-icons iicon t2s" @click="mute(false)" v-if="Config.features.speech && muted == true">volume_off</i>
            </div>
            <div class="wrapper" v-else>
                <i class="material-icons iicon recording" @click="microphone(false)">mic</i>
                <input class="queryform" :placeholder="speech" readonly>   
            </div>
        </div>

        <main class="wrapper ai-window">

            <!-- Display Welcome Message -->
            <div v-if="answers.length == 0 && online == true">
                <h1 class="title mdc-typography--headline">
                    <div class="material-icons up" v-if="!Config.features.inputBottom">arrow_upward</div>
                    {{ $t("welcome.title") }}
                    <p class="mdc-typography--body2">{{ $t("welcome.subtitle") }}</p>
                    <div class="material-icons down" v-if="Config.features.inputBottom">arrow_downward</div>
                </h1>
            </div>

            <!-- Display offline message -->
            <div v-if="answers.length == 0 && online == false">
                <h1 class="title mdc-typography--headline">
                    <div class="material-icons up">cloud_off</div>
                    <br>
                    <br>
                    {{ $t("offline.title") }}
                    <p class="mdc-typography--body2">{{ $t("offline.subtitle") }}</p>
                </h1>
            </div>

            <!-- Chat window -->
            <table v-for="a in answers" :key="a.id" class="chat-window">

                <!-- Your messages -->
                <tr>
                    <td class="bubble">{{a.result.resolvedQuery}}</td>
                </tr>

                <!-- Dialogflow messages -->
                <tr>
                    <td>

                        <!-- Bot message types / Speech -->

                        <div v-if="a.result.fulfillment.speech" class="bubble bot">
                            {{a.result.fulfillment.speech}}
                        </div>

                        <!-- Google Assistant output -->
                        <div v-for="r in a.result.fulfillment.messages" :key="r.speech">

                            <!-- Bot message types / Card -->

                            <div class="mdc-card" v-if="r.type == 'basic_card'">
                                <img :title="r.image.accessibilityText" :alt="r.image.accessibilityText" class="mdc-card__media-item" :src="r.image.url" v-if="r.image">
                                <section class="mdc-card__primary">
                                    <h1 class="mdc-card__title mdc-card__title">{{r.title}}</h1>
                                    <br>
                                    <h2 class="mdc-card__subtitle">{{r.subtitle}}</h2>
                                </section>
                                <section class="mdc-card__supporting-text">
                                    {{r.formattedText}}
                                </section>
                                <section class="mdc-card__actions" v-for="button in r.buttons" :key="button">
                                    <a class="mdc-button mdc-button--compact mdc-button--primary mdc-card__action" target="_blank" :href="button.openUrlAction.url">{{button.title}} <i class="material-icons openlink">open_in_new</i></a>
                                </section>
                            </div>

                            <!-- Bot message types / Carousel Card -->

                            <div class="slider" v-if="r.type == 'carousel_card'">
                                <carousel 
                                        :perPage="1" 
                                        :navigationEnabled="true"
                                        :paginationEnabled="false"
                                        navigationNextLabel="<button class='mdc-fab material-icons rightnav'><span class='mdc-fab__icon'>keyboard_arrow_right</span></button>"
                                        navigationPrevLabel="<button class='mdc-fab material-icons leftnav'><span class='mdc-fab__icon'>keyboard_arrow_left</span></button>"
                                        :navigationClickTargetSize="0"
                                        :loop="true">

                                    <slide v-for="item in r.items" :key="item.id">
                                        <div class="mdc-card slide">
                                            <img class="mdc-card__media-item" :src="item.image.url" v-if="item.image">
                                            <section class="mdc-card__primary">
                                                <h1 class="mdc-card__title mdc-card__title mdc-theme--primary" @click="autosubmit(item.optionInfo.key)">{{item.title}}</h1>
                                            </section>
                                            <section class="mdc-card__supporting-text">
                                                {{item.description}}
                                            </section>
                                        </div>
                                    </slide>
                                </carousel>
                            </div>

                            <!-- Bot message types / List -->

                            <div class="mdc-list-group mdc-card" v-if="r.type == 'list_card'">
                                <h3 class="mdc-list-group__subheader">{{r.title}}</h3>
                                <ul class="mdc-list mdc-list--two-line mdc-list--avatar-list" v-for="item in r.items" :key="item" @click="autosubmit(item.optionInfo.key)">
                                    <li class="mdc-list-item">
                                        <img :title="item.image.accessibilityText" :alt="item.image.accessibilityText" class="mdc-list-item__start-detail" width="56" height="56" :src="item.image.url" v-if="item.image"/>
                                        <span class="mdc-list-item__text">
                                            {{item.title}}
                                        <span class="mdc-list-item__text__secondary">{{item.description}}</span>
                                        </span>
                                    </li>
                                </ul>
                            </div>

                            <!-- Bot message types / Link Chip -->

                            <div v-if="r.type == 'link_out_chip'" class="chips">
                                <a class="suggestion link" :href="r.url" target="_blank">
                                    {{r.destinationName}} <i class="material-icons openlink">open_in_new</i>
                                </a>
                            </div>

                            <!-- Bot message types / Suggestion Chip -->

                            <div v-if="r.type == 'suggestion_chips'" class="chips">
                                <div class="suggestion" @click="autosubmit(s.title)" v-for="s in r.suggestions" :key="s">
                                    {{s.title}}
                                </div>
                            </div>

                        </div>
                    </td>
                </tr>
            </table>

            <p class="copyright" v-if="answers.length > 0" id="bottom">
                <span v-if="Config.features.about">
                    {{ $t("about.developedBy") }} <a href="https://mish.io">Ushakov</a> & <a href="https://dialogflow.com">Dialogflow</a>
                </span>
            </p>

        </main>

    </div>

    <div class="icon" :class="[displayed ? 'close':'open']" @click="displayed = !displayed" v-if="Config.features.closeable"></div>

</section>
</template>

<style lang="sass">
@import url('https://fonts.googleapis.com/css?family=Open+Sans:400,600,700')

html, body, .brand-frame
    height: 100%
    overflow: hidden

.brand-frame
    width: 100%

$color: #E0001A

#app
    position: absolute
    height: 400px
    width: 300px
    bottom: 40px
    right: 40px

.app-wrapper
    box-shadow: grey 0px 0px 22px
    display: flex
    flex-direction: column
    height: 100%

.app-title
    height: 40px
    display: flex
    justify-content: center
    align-items: center
    color: white
    font-weight: 600
    background-color: var(--mdc-theme-primary)

\:root
  --mdc-theme-primary: $color

body
    margin: 0
    font-family: 'Open Sans', sans-serif

#app.closeable
    margin-bottom: 70px

.icon
    background-repeat: no-repeat
    height: 60px
    width: 60px
    background-size: 100%
    position: absolute
    bottom: -70px
    right: 0
    &.open
        background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFAAAABQCAMAAAC5zwKfAAAABGdBTUEAALGPC/xhBQAAAAFzUkdCAK7OHOkAAABvUExURQAAAOQpP+xmdvGSnec6TvfAxvGUneY0Setba/CQnOpWaOxmdfCFkOY1SuMZMOYyRu+Ile+AjetbbO5teuhIWvGSnu59i+AAGv///+IRKeEFHv79/uQfNvGPmvWpsv709fvh5OlKXO97iPfAxvnP1G0sMa4AAAAXdFJOUwDwhn63ORj+/gtEpUmT8dxpqrws7rzgxZBfTgAAAz5JREFUWMPtmce6ozAMhWnBmJKQMh+4UAy8/zMOhBRAMi0sZjFndy/h93GXhGEsiFIa2tZTdtj+Yfwi9x5HD8KSjxh5RPHd3UsLLjJBJC/BDubpemGJVuxyPW3DBSZPZsXN4HQgbhvSXoPrkfYae85K3BPpLJmkvplskunPrkx6lclGsesMkUY82SzuaInU2cGbIe7k6Yk6HidelrfKKiW3EG/4b1VeFiLtJYqmwpk3yPuD+WNemU5U5AQj/pnyQoa5a1JERYb8loWT/YE0y7IixdUgvyajPePeYIdlnmpVeMgwDs/IGHZC1iKdIVbAARucFO4F9neW1054BoiXr0ULPOT5PK8lVuAd62MQnjAVzmsXYvlZlBV7633yuFqDssR5NZHQuijU6zVLN4JcM8ElSzJkgmo5tmgDg0SzALNnU0Uxxnmf97ndHzJgBLNUB0xIXVb1aCMO7TwPCRdsd1ZqgHX3kA0ej3Ht2Hd9toFBJeaWCs/EYEwnsjb1uJvmqmq+zRUMORgp2Oi8TtdKgemkRgx3SbkaCM4IHhsW3MZiNTBL4CDCIWTpD0DH8I4FegY5FkgMeSxQGsmxwOQ/8F8EsmOB7Ph1qI4FquP3cnQokEfIlbIBCLtnGy77AQiOKuYa9DsrLO+1mgdvPdVezNHy7SQ0t0wO7qOoi66//1WFJubCTQuw5HgXaVNzKUgSVYOH2qDHfbQULIVxOR5KCDjHQZ9RDHYfQYi1xMe2BgblK7cY3qSkmbpppIcaLOH19s7Q/GFTrCom/tBxEEhmwfxP2jg2Pgr2GZ4PwJi9u+Tf8iW+U0TptatdoHlKMmNwkom+gKJN6iTO6xqaz0hHqR57vZSRLgJGeEWO5bjjZM9mo+iraHKvfUlWJYZTaCY8ruHQQad55T0rci1OgLmtK4In/rdJfY0+QJPKm0oRpitLKFAkOJHkB0mkwhSe9/POIVYV8XcTzz5euNlL1PH2EvW8lmhu55nhYdXD/kRYqiBe2bZi33KJ8/5YbZI/wlVF2Ou6Iiw3r/TIMvE5OG0qZJPZseQkOG0ttccO0RUUpROf9nwNuMcR/BogL1F83/+9grq2FTmKPKWcyLLdhYn4C6tjzeViO/lsAAAAAElFTkSuQmCC')
    &.close
        background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFAAAABQCAMAAAC5zwKfAAAABGdBTUEAALGPC/xhBQAAAAFzUkdCAK7OHOkAAACEUExURQAAAO99iPGSneY1Suc6Tuxkc/a8wt8AGfOkreY4TOQoPuxmdeIQKOxmdvCEkfSvtu5+jOtbbOUqQOYzR+lPYeAAGuADHf309eEIIeIXL/vj5ulSZOIQKOUrQehGWeQgNvzs7vWzu/KcpvSosetfb+1vfve/xfnP1Pra3u+CjvGQnP77+0d1VWYAAAAVdFJOUwAcfpO3RDb9Df70peuGrkxtvOjb67WmN6MAAAPqSURBVFjDrZnnloIwEIWRosSClUAgdCzo+7/fTkBdNYU6f/aIy3fuZZIZMmpaS9i2vdU3dehb+KCNCXu2M04Rdd9Bo5Oxmw2E2tbqSFxBkOPK6s+cO2LaU+nRmffDrZbYVQZeruYT4voh9S64Bql3kWf6bufwzXlrapdur1iqE247xO0ZxFEQbcN3e4dvSom2id0BgaXEYTw50Vy33bmAWC/Y3/rT+6qYuMeLdjGSf8F7nudgd+EODuz88izijgG61PrZH8gdGWj7tWD2eCwQ7z8TsyPu6KAflWJ2dCeI4+wN3OApgHjzFnjgN2iCIsW+9hFK+K8PL4k7TiCOyvwsJ9KoupcJf5chE4hJeQmulYyI0f0aXEKCJRJ1/j4aXtL0WiFf7Le6ekEWU/6bJtGm4KaouqYeaKQCv6DPSy/nRJDJukjMEpGKqLp43vXOuwa/N88Dnkg9YZ51YSnx0VnomsZVrU/8NNzNl+PFt0Yg1q6xwK8kX6ww2khS7BiRuf7U4sc5+M3Efutnb2tbLKueL9fvdOLab3BRrFC81Tbycowb13f0dE1R3uRXsYc2n4tmwW/Acwau88b102+p4i1MrVQ1jDozjWvwC+slyMpI2btDDSk7UOP6Bq4JAn2pWh8r3FqibmmN61seh4xXlElLpUs03NIjwXWWpre86OCXWdJamy6UMnAN0e6XRTuwcf14sPx2qOwdgJjAlgHg5UwmAWISF8HD8x5BjiieAEjDIoB6dYVcF4KiygGpGogJWy9Bdq6yANZj2KaRapF6YYNf4BVhEpVADPK45TlGWqwCYhoCJs2hIfmkLNJ217F2UgCZXzAK+kAWTsqidq3UeNIMRfli+WV+GwROwlbX0Jp1KRD0vfz+C04fQaEi6tqMSIDQ7f/9Pi/VroNc7hranh2KgU+DxdfNr4tSjTE0ZkMIBH3M728Knq4zmUb2dmP5AqDfpDQPf+sBaFS4xuxN2z7wQEVCQWPBPYrvt6UVB2R+Uy+4x6KNBotJ6nrVnACSXyD0N+ZX8uifGu8xV23J8yTAvX7Rcxbc5Au43uDQD7hNaH4cen57XVbF8sICxEpQv/8PPybfmUqk2rKYIEG/Mv/PZdwrIvZ9ZZkSfU+sz5PoBKeK/aRHPXhn+BqQ6HQsj37PcMafHvc/h3r7NI4Xc0OCbTSGF235GYG1Xg/mrS3RVMRaT8sbTFzIeAOJCh4QD/2BB0s9Pey5HrG5bZkg9hv4UafDiPPUWSSOrU5DWKfbEBYvHXvCMXGvuTNDIuULv49W876j9p0ZyZpUZO7mQ34NmO0Mfn5PjsZuNub3Cn1jmCGqIzSNjd72W8Ufx+2wiwy8H60AAAAASUVORK5CYII=')

.wrapper.ai-window
    padding: 1rem
    height: 100%
    overflow-y: auto
    flex: 1
    background-color: #F2F2F2

.up, .down
    font-size: 32px
    background-color: white
    padding: 10px
    border-radius: 50%
    margin: 20px

.title
    vertical-align: middle
    text-align: center
    font-weight: 700
    color: rgba(0,0,0,0.7)
    margin-top: 30%

.query
    background-color: white
    box-shadow: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24)
    z-index: 999
    width: 100%
    &.bottom
        order: 1

.query > .wrapper
    display: flex
    justify-content: space-between
    align-items: center
    padding: 12px

.queryform
    border: 0
    width: 100% - 20%
    font-size: 16px
    outline: none
    color: rgba(0,0,0,0.8)
    font-weight: 500
    margin: 0 12px

    @media screen and (max-width: 320px)
        width: 100% - 35%

.iicon
    color: rgba(0,0,0,0.8)
    cursor: pointer

.recording
    color: #F44336

.chat-window
    width: 100%

.bubble
    max-width: 300px
    background-color: #E1E1E1
    padding: 16px
    border-radius: 8px
    color: rgba(0,0,0,0.7)
    float: right
    animation: msg .25s linear

.bubble.bot
    background-color: white
    float: left
    margin-right: 10px

td
    margin-top: 30px
    margin-bottom: 10px

.mdc-card
    background-color: white
    max-width: 300px
    margin-bottom: 5px
    animation: msg .45s ease-in-out

.slide
    margin: 5px
    max-width: 300px

.slider
    max-width: 300px
    margin-left: -5px

.mdc-fab
    background-color: white
    color: $color
    
.rightnav
    margin-left: -32px

    @media screen and (max-width: 720px) 
            margin-left: -35px

    @media screen and (max-width: 320px) 
            margin-left: -70px

.leftnav
    margin-right: -35px

    @media screen and (max-width: 720px)
        display: none

.openlink
    vertical-align: middle
    margin-top: -5px
    margin-left: 5px

.mdc-card__media-item
    height: auto
    width: 100%
    margin-top: -5px

.chips
    margin-left: -10px

.suggestion
    margin-top: 10px
    float: left
    margin-left: 10px
    padding: 10px
    border: 2px rgba(0,0,0,0.5) solid
    color: rgba(0,0,0,0.5)
    border-radius: 6px
    cursor: pointer
    animation: controls .25s linear

.suggestion:active
    border: 2px rgba(0,0,0,1) solid
    color: rgba(0,0,0,1)

.suggestion.link
    color: white
    background-color: $color
    border: 2px $color solid

.suggestion.link:active
    background-color: darken($color, 10%)
    border: 2px darken($color, 10%) solid

.mdc-list-item__start-detail
    border-radius: 50%

@keyframes msg
    0%
        opacity: 0
        transform: scale(0.8)
    100%
        opacity: 1
        transform: scale(1)

@keyframes controls
    0%
        transform: scaleY(0)
    100%
        transform: scaleY(1)

.copyright
    font-weight: 600
    color: rgba(0,0,0,0.8)

.copyright a
    text-decoration: none
    color: rgba(0,0,0,0.8)
    border-bottom: 2px solid transparent

.copyright a:hover
    color: $color
    border-bottom: 2px solid $color
    
</style>

<script>
import { ApiAiClient } from 'api-ai-javascript'
import Config from '../config'

const client = new ApiAiClient({ accessToken: Config.dialogflow.accessToken })

export default {
  name: 'app',
  data: function() {
    return {
      Config,
      answers: [],
      query: '',
      speech: this.$i18n.t('generic.defaultSpeech'),
      micro: false,
      muted: !Config.speech,
      displayed: !Config.features.closeable,
      online: navigator.onLine
    }
  },
  watch: {
    answers: function(val) {
      setTimeout(() => {
        document
          .querySelector('.copyright')
          .scrollIntoView({ behavior: 'smooth' })
      }, 2) // if new answers arrive, wait for render and then smoothly scroll down to .copyright selector, used as anchor
    }
  },
  methods: {
    submit() {
      client.textRequest(this.query).then(response => {
        // console.log('response', response)
        this.answers.push(response)
        this.handle(response) // <- handle the response in handle() method

        this.query = ''
        this.speech = this.$i18n.t('generic.defaultSpeech') // <- reset query and speech
      })
    },
    handle(response) {
      if (
        response.result.fulfillment.speech ||
        response.result.fulfillment.messages[0].type == 'simple_response'
      ) {
        let speech = new SpeechSynthesisUtterance(
          response.result.fulfillment.speech ||
            response.result.fulfillment.messages[0].textToSpeech
        )
        speech.voiceURI = 'native'
        speech.lang = Config.lang.speech

        if (this.muted == false) window.speechSynthesis.speak(speech) // <- Speech output if microphone is allowed
      }
    },
    autosubmit(suggestion) {
      this.query = suggestion
      this.submit()
    },
    mute(mode) {
      this.muted = mode
    },
    microphone(mode) {
      this.micro = mode
      let self = this // <- correct scope

      if (mode == true) {
        let recognition = new webkitSpeechRecognition() // <- chrome speech recognition

        recognition.interimResults = true
        recognition.lang = Config.lang.recognition
        recognition.start()

        recognition.onresult = function(event) {
          for (var i = event.resultIndex; i < event.results.length; ++i) {
            self.speech = event.results[i][0].transcript
          }
        }

        recognition.onend = function() {
          recognition.stop()
          self.micro = false
          self.autosubmit(self.speech)
        }
      }
    }
  }
}
</script>