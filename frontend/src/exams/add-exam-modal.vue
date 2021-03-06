<template>
  <b-modal v-model="modalVisible"
           :no-close-on-backdrop="true"
           id="add-exam-modal"
           hide-ok
           hide-header
           hide-cancel
           @shown="initialize"
           @hidden="hideModal"
           size="md">
    <template slot="modal-footer">
      <template v-if="unSubmitted">
        <div v-if="step < lastStep"
             style="width: 100%;
                    display: flex;
                    justify-content: space-between">
          <div>
            <b-button @click="clickCancel">Cancel</b-button>
          </div>
          <div style="display: inline">
            <b-button v-if="step > 1"
                      class="btn-secondary mr-2"
                      @click="clickBack">Back</b-button>
            <b-button v-if="button.nextDisabled"
                      :class="button.nextClass"
                      @click="setWarning"
                      id="add_exam_next">Next</b-button>
            <b-button v-else
                      :class="button.nextClass"
                      @click="clickNext"
                      id="add_exam_next">Next</b-button>
          </div>
        </div>
        <div v-else-if="step == lastStep"
             style="display: flex;
                    justify-content: space-between;
                    width: 100%">
          <div style="display: inline">
            <b-button class="btn-secondary"
                      @click="clickCancel">Cancel</b-button>
          <b-button class="btn-warning"
                    @click="logAnother">Start Again</b-button>
          </div>
          <div style="display: flex">
            <b-button v-if="errors.length > 0"
                      @click="submitMsg='You have an error on a previous step.  Click on the red tab.'"
                      class="btn-primary disabled"
                      id="add_exam_submit">Submit</b-button>
            <b-button v-else
                      class="btn-primary"
                      @click.once="submit"
                      id="add_exam_submit">Submit</b-button>
          </div>
        </div>
      </template>
      <template v-if="!unSubmitted">
        <div style="display: flex; justify-content: flex-start">
          <b-button class="btn-secondary"
                    @click="clickCancel"
                    id="add_exam_submit_close">Close</b-button>
        </div>
      </template>
    </template>

    <b-nav tabs class="mb-3">
      <b-nav-item v-for="i in tabs"
                  :key="'tab '+i.title"
                  :active="tabs[step-1].title==i.title"
                  @click="clickTab(i.step)">
        <span :style="tabWarning(i)">{{ i.title }}
        <font-awesome-icon v-if="tabValidate(i.step)"
                           icon="check"
                           class="m-0 p-0"
                           style="font-size: .8rem; color: green"/>
        </span>
      </b-nav-item>
    </b-nav>
    <template v-if="unSubmitted">
      <AddExamFormController v-if="step <= (lastStep - 1)"  />
      <AddExamFormConfirm v-if="step==lastStep" :submitMsg="submitMsg" />
    </template>
    <template v-if="!unSubmitted">
      <div v-if="status==='unknown' "
           class="loader" style="margin-top: auto"></div>
      <div v-if="status==='success'">
        <b-container>
          <b-row align-v="center"
                 align-h="center"
                 align-content="center">
            <b-col>
              <p><h5>Success.  Exam Details Added.</h5></p>
              <p><b-button @click="logAnother" class="btn-primary">Log Another Exam</b-button></p>
            </b-col>
          </b-row>
        </b-container>
      </div>
      <div v-if="status==='failed'">
        <b-container>
          <b-row align-v="center"
                 align-h="center"
                 align-content="center">
            <b-col>
              <p class="message-text">Something Went Wrong</p>
              <p><b-button @click="tryAgain">Try Again</b-button></p>
            </b-col>
          </b-row>
        </b-container>
      </div>
    </template>
  </b-modal>
</template>

<script>
  import { mapActions, mapGetters, mapMutations, mapState } from 'vuex'
  import AddExamFormController from './add-exam-form-controller'
  import AddExamFormConfirm from './add-exam-form-confirm'
  import moment from 'moment'

  export default {
    name: 'AddExamModal',
    components: { AddExamFormController, AddExamFormConfirm },
    data() {
      return ( {
        submitMsg: '',
        unSubmitted: true,
        status: 'unknown',
      } )
    },
    computed: {
      ...mapGetters({
        steps: 'add_modal_steps',
        button: 'add_exam_modal_navigation_buttons',
      }),
      ...mapState({
        exam: 'capturedExam',
        examTypes: 'examTypes',
        addExamModal: 'addExamModal',
        tab: 'captureITAExamTabSetup',
        user: 'user',
        module: 'addExamModule',
      }),
      lastStep() {
        if (this.addExamModal.setup === 'challenger') {
          return 3
        }
        return 4
      },
      errors() {
        if (this.tab.errors) {
          return this.tab.errors
        } else {
          this.submitMsg = ''
          return []
        }
      },
      modalVisible: {
        get() {
          return this.addExamModal.visible
        },
        set(e) {
          this.setAddExamModalSetting(e)
        }
      },
      step() {
        if (this.tab && this.tab.step) {
          return this.tab.step
        }
        return 1
      },
      tabs() {
        if (this.steps && Array.isArray(this.steps)) {
          return this.steps.slice(0, this.tab.highestStep)
        }
      },
    },
    methods: {
      ...mapActions([ 'clearChallengerBooking', 'clickAddExamSubmit', 'getBookings', 'getExams', ]),
      ...mapMutations([
        'captureExamDetail',
        'resetCaptureForm',
        'resetAddExamModal',
        'resetCaptureTab',
        'setAddExamModalSetting',
        'updateCaptureTab',
      ]),
      clickBack() {
        let step = this.step - 1
        this.updateCaptureTab({ step })
      },
      clickCancel() {
        this.resetModal()
        this.resetAddExamModal()
      },
      clickNext() {
        let step = this.step + 1
        this.updateCaptureTab({ step })
        if (step > this.tab.highestStep) {
          this.updateCaptureTab({ highestStep: step })
        }
      },
      clickTab(e) {
        this.updateCaptureTab({ step: e })
      },
      filterKeyPress(e) {
        if (e.keyCode === 13) {
          e.preventDefault()
        }
      },
      hideModal() {
        document.removeEventListener('keydown', this.filterKeyPress)
      },
      initialize() {
        document.addEventListener('keydown', this.filterKeyPress)
        let { setup } = this.addExamModal
        let reset = () => {
          this.unSubmitted = true
          this.submitMsg = ''
          this.status = 'unknown'
          this.captureExamDetail({ key: 'exam_method', value: 'paper' })
        }
        switch (setup) {
          case 'challenger':
            if (this.addExamModal.fromCalendar) {
              this.captureExamDetail({ key: 'offsite_location', value: this.module.challengerBooking.resource })
              this.captureExamDetail({ key: 'exam_time', value: this.module.challengerBooking.start })
              this.captureExamDetail({ key: 'expiry_date', value: this.module.challengerBooking.start })
              if (this.module.challengerBooking.invigilator) {
                this.captureExamDetail({ key: 'invigilator', value: this.module.challengerBooking.invigilator })
              }
              this.$nextTick(function() { this.$root.$emit('validateform') })
              this.clearChallengerBooking()
              return
            }
            this.resetModal()
            this.captureExamDetail({ key: 'on_or_off', value: 'off' })
            reset()
            return
          case 'individual':
            this.resetModal()
            let d = new Date()
            let today = moment(d).format('YYYY-MM-DD')
            this.captureExamDetail({ key: 'exam_received_date', value: today })
            let recd = moment().add(90, 'd')
            this.captureExamDetail({ key: 'expiry_date', value: recd })
            return
          case 'group':
            this.resetModal()
            let { office_id, office_number } = this.user.office
            office_id = parseInt(office_id)
            office_number = parseInt(office_number)
            return
          case 'other':
            this.resetModal()
            this.captureExamDetail({ key: 'on_or_off', value: 'on' })
            let exp = moment().add(60, 'd')
            this.captureExamDetail({ key: 'expiry_date', value: exp })
            return
          case 'pesticide':
            this.resetModal()
            let expiry = moment().add(60, 'd')
            this.captureExamDetail({ key: 'expiry_date', value: expiry })
            return
          default:
            this.resetModal()
            this.hideModal()
            return
        }
      },
      logAnother() {
        this.resetModal()
        this.unSubmitted = true
        this.submitMsg = ''
        this.status = 'unknown'
        this.captureExamDetail({ key: 'exam_method', value: 'paper' })
        this.initialize()
      },
      resetModal() {
        this.resetCaptureForm()
        this.resetCaptureTab()
        this.unSubmitted = true
        this.submitMsg = ''
        this.status = 'unknown'
        this.captureExamDetail({ key: 'exam_method', value: 'paper' })
      },
      setWarning() {
        if (!this.errors.includes(this.step)) {
          let errors = this.errors.concat([ this.step ])
          this.updateCaptureTab({ errors })
        }
      },
      submit() {
        let { setup } = this.addExamModal
        this.unSubmitted = false
        this.submitMsg = ''
        if (setup === 'group') {
          this.clickAddExamSubmit(setup).then(resp => {
            this.status = resp
            this.getExams()
          }).catch(error => {
            console.log(error)
            this.status = error
            this.getExams()
          })
        }
        if (setup === 'challenger') {
          this.clickAddExamSubmit(setup).then(resp => {
            this.status = resp
            this.getExams()
            this.getBookings()
          }).catch(error => {
            this.status = error
            this.getExams()
          })
        }
        if (setup === 'individual' || setup === 'other' || setup === 'pesticide') {
          this.clickAddExamSubmit('individual').then(resp => {
            this.status = resp
            this.getExams()
          }).catch(error => {
            this.status = error
            this.getExams()
          })
        }
      },
      tabValidate(i) {
        if (this.validated().indexOf(i) === -1) {
          return false
        }
        return true
      },
      tabWarning(i) {
        if (!Array.isArray(this.errors)) return ''
        if (this.errors.length > 0) {
          let list = []
          this.errors.forEach(error => {
            if (this.steps.some(step => step.step == error)) {
              list.push(this.steps.find(step => step.step == error)).questions
            }
          })
          if (list.includes(i)) {
            return { color: 'red' }
          }
          return ''
        }
        return ''
      },
      tryAgain() {
        this.unSubmitted = true
        this.status = 'unknown'
      },
      validated() {
        if (this.tab && this.tab.stepsValidated) {
          if (Array.isArray(this.tab.stepsValidated)) {
            return this.tab.stepsValidated
          }
          return [ this.tab.stepsValidated ]
        }
        return []
      },
    }
  }
</script>


<style>
  .message-text {
    font-size: .9rem; font-weight: 500
  }
  .tab-title-font {
    font-size: .8rem
  }
  .buttontabs {
    border-radius: 0 !important;
    border-bottom: 1px solid black;
    border-top: none;
    border-right: none;
    border-left: none;
  }
  .loader {
    position: relative;
    text-align: center;
    margin: 15px auto 35px auto;
    z-index: 9999;
    display: block;
    width: 80px;
    height: 80px;
    border: 10px solid rgba(0, 0, 0, .3);
    border-radius: 50%;
    border-top-color: #000;
    animation: spin 1s ease-in-out infinite;
    -webkit-animation: spin 1s ease-in-out infinite;
  }
  @keyframes spin {
    to {
      -webkit-transform: rotate(360deg);
    }
  }

  @-webkit-keyframes spin {
    to {
      -webkit-transform: rotate(360deg);
    }
  }

</style>
