<template>
<section id="app" :class="{ closeable: Config.features.closeable }">

    <transition name="fade">
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
    </transition>

    <transition name="fade">
    <div class="icon" :class="[displayed ? 'close':'open']"  @click="displayed = !displayed" v-if="Config.features.closeable"></div>
    </transition>

</section>
</template>

<style lang="sass">
@import url('https://fonts.googleapis.com/css?family=Open+Sans:400,600,700')

html, body, .brand-frame
    height: 100%
    overflow: hidden

.brand-frame
    width: 100%

$red: #E0001A
$grey: #F2F2F2

.fade-enter-active, .fade-leave-active
    transition: opacity 0.25s ease-out

.fade-enter, .fade-leave-to
    opacity: 0

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
  --mdc-theme-primary: $red

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
    animation: msg .25s linear
    &.open
        background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAGcAAABnCAMAAAAqn6zLAAAABGdBTUEAALGPC/xhBQAAACBjSFJNAAB6JgAAgIQAAPoAAACA6AAAdTAAAOpgAAA6mAAAF3CculE8AAACZFBMVEUAAADgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrgABrhCiPoRFfwgo/1qrP50NX96+3+9/j////98vP85Of3wcfznKbta3rlLELgARvhDCXrXW73usH++vv86evylJ/nOU3jGTDxipb+8/TrWmvgBB7xiJT62NzpSFvpS1374eT1rrbiFCzhBR7xjJj85unoQ1biDyf3v8b//PztZ3bjGDDgAx3iECj609f1sLjhBh/hCCH4w8n1r7fymKPyl6LrWGnvf4z97O72tLzqUWPnPFD4xsz5ys/zoKrzm6Xud4XrW2zlKkD3wMbpSVz+9PX0p7DsYnLqUmT+9vfwgI3kIDfhCyT73eD98PLtb37ucoDvfYriEirudYPwhZHkJTvsYHD619vsY3P729/4yc/3vcTgAhzlKT/xi5f86uztbHv4yM74xMr61NjucX/zmaPjHDP2tb3lKD7mNEnoQFP4x833u8LznqjudoT2uL/5zNH4wsjsZHT//f3ykp3taXj74OPxj5r++frpTF7mLkPvfovkIjn3vsXrV2j85+nxjZniDSbjGzL2t7750dbqT2H/+/vwhJHhByDoQlXylaD2s7vqU2XsZXX//v72srrnOEz74uX85ejudILmMEX++PniEyvlJjz+9fb3vMPveIbucH/rWWroRVjkHzYLRf77AAAALXRSTlMAAQIDBAs+bZq+3vwHRI7LBk+o9CWN7Dqx/je4Hqd09irNa/cMqSDUNOhC9Ur8+3itAAAAAWJLR0Q13rbZawAABOdJREFUaN7dmul/E0UYx9MmTdKmSZr0vu+LMoWWDuWqIIpaPIqKxapBQFDAAylQoIAUtWiLFkWpSj3qAdb7VgRPwAv+KXbTNNnjeZ7dTPKZF/7eNNnpPN/PbGZn5/k943Akp4xMp9PldGZmJNnPtrLcHm92jo/NyZeT7fW4s9LKyPUHggmCVr5gwJ+bHkheKJzPKOWHQ3kpUwoKi5i1igoLUqIUl5TaoKgqLSkWppSV24TMqrxMiFJRmRRFVWVF0hRXVXXSGMaqq1zJYWpqBSiqamuSwdTVC2IYq6+zTWloFKaoamywh2lqTgnDWHOTHUxLa4oYxlpbrDHz2lLGMNY2zwozX2Q6m1U932I06cEoIHJELfRNa1+wsKNzURfni7uXLF22fAV564jfqImaAj03rFzFdbpx9U0EqhWddQ3EhL55zS0cUOett6FdmrHnCH8823u7OKK1t6O9GmFMHdrhjjs5obv6sH7gElSDrWnr7ua07rkX6VlfY8a4sBV6/X3cSv0bkL615tdEFfKv93daYhStGYB7VxkxFcgD+sCDdjCcPxQBu1cb37CVMGbjw/YwnG+CR1Spx5Qhd22zXQznW+AI+s0JsrN5xD6G861giHItphjG9G0jAz/62PYd2u87H49rS+KZ0u7rSmDOEyTmSbZuB9r4VDxISQJTAO86d5GYpxnbjbUN9iZmRWliS1wID2cPydnL2D7175CpZdt+3UpUOIfJg7fqB/pJzm7GDg5zfuiw4Xr3kR59nKK8GCcED+cZenqpa83RkWPPPqe7+vxoxBQoFOOEYc5xmvNC7B33ovbiyBgQKDyLyYXTqHFuoRPmUb8Evl/zZzM+Pzycl604fGLDgtGTuiv9PWAof5QTgDkdlhxA8JsoEOUEYc4rIpxXwVBBFZMFZ9IDq0Q4p8BYPjX9d8PDeU0Ew1+Hg7kVjgduOi3EmYCDeRSOF26aFOJ0wsG8CicbbnojnZxshZMjgZOjcHwSOD7F6GISOCzDkSmFk+lwSuE4pXFcUjiu/919kzXfZD0/stYDaeubrPXam07Om3AwL/4+FeO8BQfz4PsDMc4ZOJgb3++Icabg6ZaF79/EOLAJEzTvR9fPZX5vi2CQfW8A2F+/YxFqmGp8F/55/EC+8B6NmX6fav0AxMTyBX3+E/mQ5Hy0iWg8exTkhMF87hyFWXvgLNG6Gb5tITg/PYUHGvp4hsD0fwJi4vmpId+OfIpG+mzj5wTnC3g48Xzb6B+0n0QCTbERAtP9JYjR+AdGPyQCGyH72VeLccw04ipq/BCzv3MOcESPsBVfE8OZgTE6f8fsV0UWGsIM7mNsisCcQaxsnV8F+G+G9e2bXYx9O41jOhDj0uC/mf1EHee7739g7MchHPMThjH4iWZ/VMs5P65c+Bm1ynnXBYRi9kdNfm+cc/GXX5WvAzMXUczEcgxj9ntN/nWMM3ws+pCfnkApv42OoRjAvzb68Qrn9/N//Bn93Nd7CaPsBPypuCA/3lhfuDx5OfbpyolBmPHX3/+MM0pIiROul0z+C+nCf1evjTFaSL2ErP8ICK3/0PWsZNVKVFFb0lHUnFUbWUOVVG+UVj+VVg+WVt+WVq+Xdv7AIes8hUPa+RBp510css7vqJJzHkmVnPNVquScF1Ml5/xbVFLO88Uk43yiRqLnLa8D1DVOHkBJ6tIAAAAldEVYdGRhdGU6Y3JlYXRlADIwMTgtMDMtMjdUMTY6Mjg6NDMrMDA6MDCKWkQtAAAAJXRFWHRkYXRlOm1vZGlmeQAyMDE4LTAzLTI3VDE2OjI4OjQzKzAwOjAw+wf8kQAAAABJRU5ErkJggg==')
    &.close
        background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFAAAABQCAMAAAC5zwKfAAAABGdBTUEAALGPC/xhBQAAAAFzUkdCAK7OHOkAAACEUExURQAAAO99iPGSneY1Suc6Tuxkc/a8wt8AGfOkreY4TOQoPuxmdeIQKOxmdvCEkfSvtu5+jOtbbOUqQOYzR+lPYeAAGuADHf309eEIIeIXL/vj5ulSZOIQKOUrQehGWeQgNvzs7vWzu/KcpvSosetfb+1vfve/xfnP1Pra3u+CjvGQnP77+0d1VWYAAAAVdFJOUwAcfpO3RDb9Df70peuGrkxtvOjb67WmN6MAAAPqSURBVFjDrZnnloIwEIWRosSClUAgdCzo+7/fTkBdNYU6f/aIy3fuZZIZMmpaS9i2vdU3dehb+KCNCXu2M04Rdd9Bo5Oxmw2E2tbqSFxBkOPK6s+cO2LaU+nRmffDrZbYVQZeruYT4voh9S64Bql3kWf6bufwzXlrapdur1iqE247xO0ZxFEQbcN3e4dvSom2id0BgaXEYTw50Vy33bmAWC/Y3/rT+6qYuMeLdjGSf8F7nudgd+EODuz88izijgG61PrZH8gdGWj7tWD2eCwQ7z8TsyPu6KAflWJ2dCeI4+wN3OApgHjzFnjgN2iCIsW+9hFK+K8PL4k7TiCOyvwsJ9KoupcJf5chE4hJeQmulYyI0f0aXEKCJRJ1/j4aXtL0WiFf7Le6ekEWU/6bJtGm4KaouqYeaKQCv6DPSy/nRJDJukjMEpGKqLp43vXOuwa/N88Dnkg9YZ51YSnx0VnomsZVrU/8NNzNl+PFt0Yg1q6xwK8kX6ww2khS7BiRuf7U4sc5+M3Efutnb2tbLKueL9fvdOLab3BRrFC81Tbycowb13f0dE1R3uRXsYc2n4tmwW/Acwau88b102+p4i1MrVQ1jDozjWvwC+slyMpI2btDDSk7UOP6Bq4JAn2pWh8r3FqibmmN61seh4xXlElLpUs03NIjwXWWpre86OCXWdJamy6UMnAN0e6XRTuwcf14sPx2qOwdgJjAlgHg5UwmAWISF8HD8x5BjiieAEjDIoB6dYVcF4KiygGpGogJWy9Bdq6yANZj2KaRapF6YYNf4BVhEpVADPK45TlGWqwCYhoCJs2hIfmkLNJ217F2UgCZXzAK+kAWTsqidq3UeNIMRfli+WV+GwROwlbX0Jp1KRD0vfz+C04fQaEi6tqMSIDQ7f/9Pi/VroNc7hranh2KgU+DxdfNr4tSjTE0ZkMIBH3M728Knq4zmUb2dmP5AqDfpDQPf+sBaFS4xuxN2z7wQEVCQWPBPYrvt6UVB2R+Uy+4x6KNBotJ6nrVnACSXyD0N+ZX8uifGu8xV23J8yTAvX7Rcxbc5Au43uDQD7hNaH4cen57XVbF8sICxEpQv/8PPybfmUqk2rKYIEG/Mv/PZdwrIvZ9ZZkSfU+sz5PoBKeK/aRHPXhn+BqQ6HQsj37PcMafHvc/h3r7NI4Xc0OCbTSGF235GYG1Xg/mrS3RVMRaT8sbTFzIeAOJCh4QD/2BB0s9Pey5HrG5bZkg9hv4UafDiPPUWSSOrU5DWKfbEBYvHXvCMXGvuTNDIuULv49W876j9p0ZyZpUZO7mQ34NmO0Mfn5PjsZuNub3Cn1jmCGqIzSNjd72W8Ufx+2wiwy8H60AAAAASUVORK5CYII=')

.wrapper.ai-window
    padding: 1rem
    height: 100%
    overflow-y: auto
    flex: 1
    background-color: $grey

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
    border-top: 1px solid darkgrey
    background-color: $grey
    z-index: 999
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
    background: transparent
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
    min-width: 50px
    background-color: $red
    color: white
    padding: 16px
    border-radius: 40px 40px 0
    float: right
    animation: msg .25s linear

.bubble.bot
    color: black
    background-color: white
    float: left
    border-radius: 40px 40px 40px 0
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
    color: $red
    
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
    background-color: $red
    border: 2px $red solid

.suggestion.link:active
    background-color: darken($red, 10%)
    border: 2px darken($red, 10%) solid

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
    color: $red
    border-bottom: 2px solid $red
    
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