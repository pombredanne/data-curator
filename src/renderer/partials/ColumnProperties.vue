  <template>
  <form class="navbar-form form-horizontal" id="columnProperties">
    <div class="form-group-sm row container-fluid">
      <div class="propertyrow" v-for="(formprop, index) in formprops" :key="index">
        <label v-tooltip.left="tooltip(formprop.tooltipId)" class="control-label col-sm-3" :for="formprop.label">
          {{formprop.label}}
        </label>
        <component :is="formprop.tooltipView"/>
        <template v-if="typeof formprop.type && formprop.type === 'dropdown'">
          <select v-if="formprop.key==='type'" :value="getTypeProperty" v-model="typeProperty" @input="setTypeProperty($event.target.value)" :id="formprop.key" class="form-control input-sm col-sm-9">
            <option v-for="option1 in typeValues" :key="option1" v-bind:value="option1">
              {{ option1}}
            </option>
          </select>
          <select v-if="formprop.key==='format'" :value="getFormatProperty" v-model="formatProperty" @input="setFormatProperty($event.target.value)" id="format" :disabled="isDropdownFormatDisabled" class="form-control input-sm col-sm-9">
            <option v-for="option2 in formatValues" :key="option2" v-bind:value="option2">
              {{ option2}}
            </option>
          </select>
        </template>
        <div v-else-if="formprop.key === 'constraints'" id="constraints" class="col-sm-9">
          <div class="input-group row" v-for="option in constraintValues" :key="option">
            <input type="checkbox" :id="option" :checked="getConstraintCheck(option)" @click="setConstraintCheck(option, $event.target)"></input>
            <label :for="option" class="form-control-static">{{option}}</label>
            <template v-if="!isBooleanConstraint(option) && getConstraintCheck(option)">
              <input type="text" :class="{ 'form-group-sm constraint-text': true,'validate-danger': errors.has(option) }" :value="getConstraintValue(option)" @input="setConstraintValue(option, $event.target.value)" v-validate="constraintValidationRules(option)" :name="option"/>
            </template>
            <div v-show="errors.has(option) && removeConstraint(option)" class="row help validate-danger">
              {{ errors.first(option)}}
            </div>
          </div>
        </div>
        <input v-else-if="formprop.key === 'name'" :disabled="formprop.isDisabled" :value="getNameProperty" @input="setProperty(formprop.key, $event.target.value)" type="text" class="form-control label-sm col-sm-9" :id="formprop.key" />
        <input v-else :disabled="formprop.isDisabled" :value="getProperty(formprop.key)" @input="setProperty(formprop.key, $event.target.value)" type="text" class="form-control label-sm col-sm-9" :id="formprop.key" />
      </div>
    </div>
  </div>
</form>
</template>
<script>
import SideNav from './SideNav'
import {
  mapMutations,
  mapGetters
} from 'vuex'
import AsyncComputed from 'vue-async-computed'
import VueRx from 'vue-rx'
import Vue from 'vue'
import {
  Subscription
} from 'rxjs/Subscription'
import {
  allTablesAllColumnNames$
} from '@/rxSubject.js'
import {
  HotRegister
} from '@/hot.js'
import ColumnTooltip from '../mixins/ColumnTooltip'
import ValidationRules from '../mixins/ValidationRules'
Vue.use(VueRx, {
  Subscription
})
Vue.use(AsyncComputed)
export default {
  extends: SideNav,
  name: 'column',
  mixins: [ValidationRules, ColumnTooltip],
  props: ['cIndex'],
  data() {
    return {
      typeValues: ['string', 'number', 'integer', 'boolean', 'object', 'array', 'date', 'time', 'datetime', 'year', 'yearmonth', 'duration', 'geopoint', 'geojson', 'any'],
      typeProperty: '',
      formatProperty: '',
      constraintInputKeyValues: {},
      allTablesAllColumnsNames: {},
      // TODO: setup args so clear for constraints only
      debounceSetConstraints: _.debounce(this.pushColumnProperty, 300, {
        'leading': true,
        'trailing': false
      }),
      formprops: [{
        label: 'Name*',
        key: 'name',
        tooltipId: 'tooltip-column-name',
        tooltipView: 'tooltipColumnName',
        isDisabled: true
      },
      {
        label: 'Title',
        key: 'title',
        tooltipId: 'tooltip-column-title',
        tooltipView: 'tooltipColumnTitle'
      },
      {
        label: 'Description',
        key: 'description',
        tooltipId: 'tooltip-column-description',
        tooltipView: 'tooltipColumnDescription'
      },
      {
        label: 'Type',
        key: 'type',
        tooltipId: 'tooltip-column-type',
        tooltipView: 'tooltipColumnType',
        type: 'dropdown'
      },
      {
        label: 'Format',
        key: 'format',
        tooltipId: 'tooltip-column-format',
        tooltipView: 'tooltipColumnFormat',
        type: 'dropdown'
      },
      {
        label: 'Constraints',
        key: 'constraints',
        tooltipId: 'tooltip-column-constraints',
        tooltipView: 'tooltipColumnConstraints',
        type: 'checkbox'
      },
      {
        label: 'RDF Type',
        key: 'rdfType',
        tooltipId: 'tooltip-column-rdfType',
        tooltipView: 'tooltipColumnRdfType',
        type: 'url'
      }
      ],
      formats: {
        'string': ['email', 'uri', 'binary', 'uuid', 'default'],
        'number': ['default'],
        'integer': ['default'],
        'boolean': ['default'],
        'object': ['default'],
        'array': ['default'],
        'date': ['any', 'pattern', 'default'],
        'time': ['any', 'pattern', 'default'],
        'datetime': ['any', 'pattern', 'default'],
        'year': ['default'],
        'yearmonth': ['default'],
        'duration': ['default'],
        'geopoint': ['array', 'object', 'default'],
        'geojson': ['topojson', 'default'],
        'any': ['default']
      },
      constraints: {
        'string': ['required', 'unique', 'minLength', 'maxLength', 'pattern', 'enum'],
        'number': ['required', 'unique', 'minimum', 'maximum', 'pattern', 'enum'],
        'integer': ['required', 'unique', 'minimum', 'maximum', 'pattern', 'enum'],
        'boolean': ['required', 'enum'],
        'object': ['required', 'unique', 'minLength', 'maxLength', 'enum'],
        'array': ['required', 'unique', 'minLength', 'maxLength', 'enum'],
        'date': ['required', 'unique', 'minimum', 'maximum', 'enum'],
        'time': ['required', 'unique', 'minimum', 'maximum', 'enum'],
        'datetime': ['required', 'unique', 'minimum', 'maximum', 'enum'],
        'year': ['required', 'unique', 'minimum', 'maximum', 'enum'],
        'yearmonth': ['required', 'unique', 'minimum', 'maximum', 'pattern', 'enum'],
        'duration': ['required', 'unique', 'minimum', 'maximum', 'enum'],
        'geopoint': ['required', 'unique', 'enum'],
        'geojson': ['required', 'unique', 'minLength', 'maxLength', 'enum'],
        'any': ['required', 'unique', 'enum']
      },
      constraintBooleanBindings: ['required', 'unique']
    }
  },
  asyncComputed: {
    getTypeProperty: {
      async get() {
        // console.log('getting type')
        let hotId = await this.currentHotId()
        // let hotId = this.activeCurrentHotId
        // console.log(`hot id in getType is ${hotId}`)
        let getter = this.getter(hotId, 'type')
        let property = this.getHotColumnProperty(getter)
        if (!property) {
          // console.log(`no get type, so setting default`)
          this.pushColumnProperty(this.setter(hotId, 'type', 'any'))
          property = 'any'
        }
        // view doesn't always re-render
        this.typeProperty = property
        this.$forceUpdate()
        return property
      },
      watch() {
        // ensure getter changes for tabs and columns
        let temp = this.getActiveTab
        let temp2 = this.cIndex
      }
    },
    getFormatProperty: {
      async get() {
        console.log('getting format')
        let hotId = this.activeCurrentHotId
        let getter = this.getter(hotId, 'format')
        let property = this.getHotColumnProperty(getter)
        if (!property) {
          this.pushColumnProperty(this.setter(hotId, 'format', property))
          property = 'default'
        }
        // console.log('got selectFormat property')
        this.formatProperty = property
        this.$forceUpdate()
        return property
      },
      watch() {
        let temp = this.getActiveTab
        let temp2 = this.cIndex
      }
    }
  },
  methods: {
    ...mapMutations([
      'pushColumnProperty'
    ]),
    isBooleanConstraint: function(option) {
      return this.constraintBooleanBindings.indexOf(option) > -1
    },
    setTypeProperty: async function(value) {
      // console.log('setting type...')
      // this.pushColumnProperty(this.setter(this.activeCurrentHotId, 'type', value))
      this.pushColumnProperty(this.setter(this.activeCurrentHotId || this.currentHotId(), 'type', value))
      this.typeProperty = value
      return value
    },
    setFormatProperty: function(value) {
      // console.log('about to set format...')
      let hotId = this.activeCurrentHotId
      this.pushColumnProperty(this.setter(hotId, 'format', value))
      this.formatValue = value
    },
    getProperty: function(key) {
      let hotId = this.activeCurrentHotId
      // console.log(`getting for ${key} and hot ${hotId}`)
      let getter = this.getter(hotId, key)
      // console.log('getter is')
      // console.log(getter)
      let property = this.getHotColumnProperty(getter)
      return property
    },
    setProperty: function(key, value) {
      // console.log(`setting for key ${key}...`)
      this.pushColumnProperty(this.setter(this.activeCurrentHotId, key, value))
    },
    getter: function(hotId, key) {
      let object = {
        hotId: hotId,
        columnIndex: this.cIndex,
        key: key
      }
      return object
    },
    setter: function(hotId, key, value) {
      let object = {
        hotId: hotId,
        columnIndex: this.cIndex,
        key: key,
        value: value
      }
      return object
    },
    getConstraintCheck: function(key) {
      console.log(`checking constraint: ${key}`)
      return _.has(this.constraintInputKeyValues, key)
    },
    // constraintText(option) {
    //   console.log(`constraint text option is: ${option}`)
    //   console.log(option)
    //   return _.has(this.constraintInputKeyValues, option)
    // },
    // toggleTextNode: function(checkedInput) {
    //   let textNode = checkedInput.parentNode.querySelector('.constraint-text')
    //   if (textNode) {
    //     textNode.style.display = checkedInput.checked ? 'inline-block' : 'none'
    //   }
    // },
    setConstraintCheck: function(key, target) {
      let isChecked = target.checked
      console.log(`is target checked: ${target.checked}`)
      if (!isChecked) {
        _.unset(this.constraintInputKeyValues, key)
      } else if (this.constraintBooleanBindings.indexOf(key) > -1) {
        this.constraintInputKeyValues[key] = isChecked
      } else {
        let getter = this.getter(this.activeCurrentHotId, key)
        let currentValue = this.getConstraint(getter) || ''
        this.constraintInputKeyValues[key] = currentValue
      }
      this.pushConstraintInputKeyValues()
      this.$forceUpdate()
    },
    getConstraintValue: function(key) {
      let property = this.constraintInputKeyValues[key]
      return property
    },
    removeConstraint: function(key) {
      this.constraintInputKeyValues[key] = ''
      this.pushConstraintInputKeyValues()
      return true
    },
    setConstraintValue: function(key, value) {
      this.constraintInputKeyValues[key] = value
      this.pushConstraintInputKeyValues()
    },
    pushConstraintInputKeyValues: function() {
      console.log('pushing contraint input key values...')
      this.debounceSetConstraints(this.setter(this.activeCurrentHotId, 'constraints', this.constraintInputKeyValues))
    },
    constraintValidationRules: function(option) {
      if (_.indexOf(['minLength', 'maxLength'], option) > -1) {
        return 'numeric'
      } else if (_.indexOf(['minimum', 'maximum'], option) > -1) {
        return this.validationRules(this.typeProperty)
      } else {
        // console.log('No validation rules to apply this constraint')
        return ''
      }
    },
    updateConstraintInputKeyValues: function() {
      // console.log('update constraint checked....')
      let hotId = this.activeCurrentHotId
      let getter = this.getter(hotId, 'constraints')
      let constraints = this.getHotColumnProperty(getter)
      this.constraintInputKeyValues = constraints || {}
    },
    updateAllTablesAllColumnsNames: function(update) {
      this.allTablesAllColumnsNames = update || {}
    }
  },
  watch: {},
  computed: {
    ...mapGetters([
      'getActiveTab', 'getHotColumnProperty', 'getConstraint', 'getAllHotTablesColumnNames'
    ]),
    getNameProperty() {
      let allColumns = this.allTablesAllColumnsNames[this.activeCurrentHotId] || []
      console.log('all columns for this tab are:')
      console.log(allColumns)
      return allColumns[this.cIndex] || ''
    },
    formatValues() {
      // console.log('updating format values for type')
      let property = this.typeProperty
      return this.formats[property]
    },
    constraintValues() {
      // console.log('updating constraint values for type')
      let property = this.typeProperty
      this.updateConstraintInputKeyValues()
      return this.constraints[property]
    },
    isDropdownFormatDisabled() {
      return !this.formatValues ? false : this.formatValues.length < 2
    }
    // selectFormat: {
    //   get: function() {
    //     console.log('about to get select format...')
    //     let hotId = this.activeCurrentHotId
    //     let getter = this.getter(hotId, 'format')
    //     let property = this.getHotColumnProperty(getter)
    //     if (!property) {
    //       property = 'default'
    //       this.pushColumnProperty(this.setter(hotId, 'format', property))
    //     }
    //     // console.log('got selectFormat property')
    //     return property
    //   },
    //   set: function(value) {
    //     // console.log('about to set format...')
    //     let hotId = this.activeCurrentHotId
    //     this.pushColumnProperty(this.setter(hotId, 'format', value))
    //   }
    // },
  },
  created: function() {
    // console.log('created...')
  },
  mounted: function() {
    let vueUpdateAllTablesAllColumnsNames = this.updateAllTablesAllColumnsNames
    this.$subscribeTo(allTablesAllColumnNames$, function(result) {
      console.log(`names in subscription`)
      vueUpdateAllTablesAllColumnsNames(result)
    })
    allTablesAllColumnNames$.next(this.getAllHotTablesColumnNames())
  },
  destroyed: function() {
    // console.log('panel destroyed')
  }
}
</script>
<style lang="styl" scoped>
@import '~static/css/columnprops'
</style>
<style lang="styl" scoped>
@import '~static/css/validationrules'
</style>
<style lang="styl" scoped>
@import '~static/css/tooltip'
</style>
