<template> 
    <ion-header>
        <ion-toolbar>
        <ion-title>New reading</ion-title>
        <ion-buttons slot="end">
            <ion-button @click="closeModal">Close</ion-button>
        </ion-buttons>
        </ion-toolbar>
    </ion-header>
    <ion-content class="ion-padding">
        <ion-item>
          <ion-label 
            position="stacked"
            v-t="'AddReading.label-value'"
            :color="submitted && v$.value.$error ? 'danger' : 'black'"
          ></ion-label>
          <ion-input required v-model="value" type="text" ></ion-input>
          <ion-text color="danger" v-if="submitted && v$.value.$error">
            <small>
              <!-- Iterate over errors and display the first error message -->
              {{ v$.value.$errors[0].$message }}
            </small>
          </ion-text>
        </ion-item>
        <ion-item>
          <ion-label 
            position="stacked"
            v-t="'AddReading.label-date'"
            :color="submitted && v$.date.$error ? 'danger' : 'black'"
          ></ion-label>
          <ion-input required v-model="date" type="date"></ion-input>
          <ion-text color="danger" v-if="submitted && v$.date.$error">
            <small>
              <!-- Iterate over errors and display the first error message -->
              {{ v$.date.$errors[0].$message }}
            </small>
          </ion-text>
        </ion-item>
        <ion-item>
          <ion-label position="stacked" v-t="'AddReading.label-comment'"></ion-label>
          <ion-input v-model="comment" type="text"></ion-input>
        </ion-item>
        <ion-button expand="block" v-t="'AddReading.button-save'" @click="save"></ion-button>
    </ion-content>
</template>

<script>
import { 
    IonContent, 
    IonButton,
    IonHeader, 
    IonToolbar, 
    IonTitle, 
    IonButtons, 
    IonItem, 
    IonLabel, 
    IonInput,
    IonText,
    modalController,
} from '@ionic/vue';
import log from 'loglevel';

import { ref } from 'vue';
import { useI18n } from 'vue-i18n';
import useVuelidate from "@vuelidate/core";
import repo from "../db/repo/readings";
import { useAppStore } from '../stores/app';
import useSQLite from "../composables/useSQLite";
import { useRouter  } from 'vue-router';
import { storeToRefs } from 'pinia';
import { required, helpers, minValue } from "@vuelidate/validators";

const name="NewReading";
const { withMessage } = helpers;
const LOG = `[component|${name}]`;

// date difference function
function dateDiffInDays(a, b) {
  a= new Date(a);
  b= new Date(b);
  const _MS_PER_DAY = 1000 * 60 * 60 * 24;
  // Discard the time and time-zone information.
  const utc1 = Date.UTC(a.getFullYear(), a.getMonth(), a.getDate());
  const utc2 = Date.UTC(b.getFullYear(), b.getMonth(), b.getDate());

  return Math.floor((utc2 - utc1) / _MS_PER_DAY);
}


export default({
  name,
  components: {
    IonContent, 
    IonButton,
    IonHeader, 
    IonToolbar, 
    IonTitle, 
    IonButtons, 
    IonItem, 
    IonLabel,
    IonText,
    IonInput,
  },
  props:{
    currMeterId: {
      type: String,
      required: true
    },
    previousReading: {
      type: Object,
      required: true
    },
    currMeterInitVal:{
      type: String,
      required: true
    }
  },
  setup(props) {
    log.debug(LOG, "setup");

    const router = useRouter();
    const store = useAppStore();
    const {run} = useSQLite();
    const { shouldReloadData } = storeToRefs(store);

    //const isModalOpen = ref(props.isVisibleForm);
    const isDateOpen = ref(false);
    const { t } = useI18n();
    const value = ref('');
    const now = ref(new Date());
    const nowDate = now.value.toISOString().split('T')[0];
    const date = ref(nowDate);
    const comment = ref('');
    const average = ref(0.0);
    const submitted = ref(false);
    const increaseDates = ref(false);
    const increaseValues = ref(false);

    const dateInput = () => {
      log.debug("dateInput");
      isDateOpen.value = true;
    }; 
    
    const closeModal = () => {
      modalController.dismiss();
    };

    // Custom validator to check if the date is not in the past
    const isNotValid = (dat) => {
      if (!dat) return true; // If no value, return true to satisfy required validator first
      if(props.previousReading){
        const previousReadingDate = new Date(props.previousReading.date);
        const selectedDate = new Date(dat);
        return selectedDate > previousReadingDate;
      }
      return true
    };
    //input validation
    const rules = {
      value: {
        required: withMessage(t("AddReading.msg-value-required"), required),
      },
      date: {
        required: withMessage(t("AddReading.msg-date-required"), required),
        // minValue: withMessage(
        //   t("AddReading.msg-date-min"),
        //   minValue(new Date(props.previousReading? props.previousReading.date: null))
        // ),
      },
    };
    if(increaseValues.value){
      rules.value.minValue= withMessage(
        t("AddReading.msg-value-min"),
        minValue(props.previousReading? props.previousReading.value: props.currMeterInitVal? props.currMeterInitVal: 0)
      )
    }
    if(increaseDates.value){
      rules.date.minValue= withMessage(
        t("AddReading.msg-date-min"),
        isNotValid      )
    }
    const v$ = useVuelidate(rules, { value, date }, { $autoDirty: true });

    const save = async () => {
      const $v = v$.value;
      const reading = {
        value: value.value,
        date: date.value,
      };
      log.debug(LOG,"previousReading", props.previousReading);
      submitted.value = true;
      $v.$touch();
      var valid = await $v.$validate();
      log.log(LOG, "saving", reading, $v);
      if (!valid) {
        log.log(LOG, "invalid");
        return;
      }
      //get the array of previousReading's ids   
      
      //check if the previous reading exist
      
        
      if (props.previousReading) {
        const previousReadingId= props.previousReading.id;
        log.debug(LOG,"previousReadingsId", { prev: previousReadingId, 
          prevDate: new Date(date.value), date:new Date(props.previousReading.date).getDate(),
        });
        //calculate the average
        const a = date.value;
        const b = props.previousReading.date;
        
        const result = dateDiffInDays(b, a);
        
        log.debug(LOG,"days between dates", a);
        log.debug(LOG,"value utc1", b);
        if(a === b){
          average.value = 0;
        }
        else{
          average.value = ((
          value.value - props.previousReading.value) / 
            result).toFixed(3);
        }
        log.debug(LOG,"calculatedAverage", average.value);
        
      }
      
      //add to db   
      const { add } = repo;
      try {
        await run(
          add({
            value: value.value,
            date: date.value,
            comment: comment.value,
            average: average.value,
            meter_id: props.currMeterId,
          })
        );
        shouldReloadData.value = true;
        //router.push('/usage');
        router.replace(`/usage/${props.currMeterId}`)
        closeModal();
      } catch (ex) {
        log.error(ex);
      }

      log.debug("saved");
    };
    
    

    return {
      submitted,
      v$,
      save,
      value,
      date,
      comment,
      closeModal,
      //isModalOpen,
      nowDate,
      isDateOpen,
      dateInput,
    };
  }
});
</script>