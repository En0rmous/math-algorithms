<template>
  <div class="input-form_middle">
    <div class="input-form_middle__item">
      <label>Шаг между узловыми точками (h):</label>
      <input type="text" class="input-form__item-header" v-model.number="diff">
    </div>
    <div class="input-form_middle__item">
      <label>Количество точек:</label>
      <input type="text" class="input-form__item-header" v-model.number="pointsAmmount">
    </div>
    <div class="input-form_middle__item">
      <label>Начальная точка:</label>
      <input type="text" class="input-form__item-header" v-model.number="start">
    </div>
    <div class="input-form_middle__item">
      <label>Точка:</label>
      <input type="text" class="input-form__item-header" v-model.number="point">
    </div>
    <div style="margin-top: 40px">
      <el-button
        type="warning"
        v-if="countedResultL.length === 0"
        @click="init"
      >
        Заполнить начальные данные
      </el-button>

      <el-button
        type="warning"
        v-else
        @click="reset"
      >
        Очистить поля
      </el-button>
    </div>
    <div style="margin: 40px 0">
      <div>
        <div v-if="fields.length > 0" class="input-form">
          <input type="text" value="i" disabled class="input-form__item input-form__item_disabled">
          <input type="text" value="X" disabled class="input-form__item input-form__item_disabled">
          <input type="text" value="Y" disabled class="input-form__item input-form__item_disabled">
        </div>
        <div v-for="(field, index) in fields" :key="index" class="input-form">
          <input type="text" :value="index" disabled class="input-form__item input-form__item_disabled">
          <input type="text" v-model.number="field.valueX" class="input-form__item">
          <input type="text" v-model.number="field.valueY" class="input-form__item">
        </div>
      </div>
    </div>

    <el-button
      v-if="checkFields && countedResultL.length === 0"
      type="warning"
      @click="countResultsL"
    >
      Рассчитать
    </el-button>

    <div v-if="resultsL.length" class="input-form_middle">
      <h4>Начальные уравнения</h4>
      <div class="initial_eq">
        <div v-for="(value, index) in resultsL" :key="`res-${index}`" class="initial-eq__item">
          <vue-mathjax v-if="value != null" :formula="`$$l_${index} = ${fields[index].valueY} * {${value.numerator.join('')} \\over${value.denominator.join('')}} = ${fields[index].valueY} *  {${countedResultL[index].numerator} \\over${countedResultL[index].denominator}}= ${answerL[index].replace(/([-\d]*)\/([-\d]*)/g, '\\frac{$1}{$2}')} $$`"></vue-mathjax>
        </div>
      </div>
      <h4>График интерполяционного многочлена Лагранджа</h4>
      <vue-mathjax :formula="`$$ L_${fields.length - 1}(x) = ${grapg.replace(/([-\d]*)\/([-\d]*)/g, '\\frac{$1}{$2}')} $$`"></vue-mathjax>
    </div>

    <div style="display: flex; justify-content: center; margin: 20px 0px;">
      <line-chart
        v-if="renderChart"
        :func="grapg"
        :labels="fields"
        style="max-width: 500px; max-height: 500px;"
        ref="line-chart"
      />
    </div>
    <div class="input-form_middle">
      <div v-for="(item, index) in resultWithNumbers" :key="`resultWIthNum-${index}`" class="initial-eq__item">
        <vue-mathjax :formula="`$$ l_${index} (${point}) = \\frac{ ${item} }{ ${resultWithLetters[index]} } = \\frac{ ${resultsL[index].numerator.join('').replace(/x/g, point)} }{ ${resultsL[index].denominator.join('')} } = \\frac{ ${Math.round10(resultForEachArr[index].num, -3)} }{ ${Math.round10(resultForEachArr[index].denum, -3)} } = ${resultForEachArr[index].response} = ${Math.round10(resultForEachArr[index].responseDec, -3)} $$`"></vue-mathjax>
      </div>
    </div>
    <div class="input-form_middle" v-if="functionRes !== ''">
      <div class="initial-eq__item">
        <vue-mathjax :formula="`$$ L_${resultWithNumbers.length - 1} (${point}) = ${this.functionRes} = ${Math.round10(functionNumbersResultDec, -1)} $$`"></vue-mathjax>
      </div>
    </div>
    <div class="input-form_middle" v-if="functionRes !== ''" style="border: 1px solid black; padding: 10px">
      <div style="padding: 10px;">
        <vue-mathjax :formula="`$$ Ответ: L_${resultWithNumbers.length - 1} (${point}) = ${Math.round10(functionNumbersResultDec, -1)} $$`"></vue-mathjax>
      </div>
    </div>
  </div>
</template>

<script>
import algebra from 'algebra.js'
import LineChart from '../components/LineChart'
require('round10').polyfill()
// eslint-disable-next-line
import { evaluate, parse } from 'mathjs'

const getResultDecNumber = (x) => {
  let afterComma = 0
  if (Number(x) === x && x % 1 !== 0) {
    if (x.toString().indexOf(',') !== -1) {
      afterComma = countDecimals(x.replace(',', '.'))
    } else {
      afterComma = countDecimals(x)
    }
  }
  return new algebra.Fraction(x * Math.pow(10, afterComma), Math.pow(10, afterComma))
}

const countDecimals = (value) => {
  if (Math.floor(value) !== value) {
    return value.toString().split('.')[1].length || 0
  }
  return 0
}

const countFunctionInPoint = (fields, resArr) => {
  let str = ''
  for (let i = 0; i < resArr.length; i++) {
    if (i < resArr.length - 1) {
      str += `(${fields[i].valueY}) \\cdot (${Math.round10(resArr[i].responseDec, -1)}) +`
    } else {
      str += `(${fields[i].valueY}) \\cdot (${Math.round10(resArr[i].responseDec, -1)})`
    }
  }
  return str
}

const countWithPoint = (arr, point) => {
  let result = arr.map((el) => {
    let num = parse(el.numerator.join('')).evaluate({ x: point })
    let denum = parse(el.denominator.join('')).evaluate()
    let response = algebra.parse(`(${num}) / (${denum})`)
    response = response.toString()
    let responseDec = evaluate(response)
    response = response.replace(/([-\d]*)\/([-\d]*)/g, '\\frac{$1}{$2}')

    return {
      num,
      denum,
      response,
      responseDec
    }
  })
  return result
}

const makeLetterString = (arr) => {
  const numerator = []
  const denum = []
  for (let i = 0; i < arr.length; i++) {
    let numStr = ''
    let denumStr = ''
    for (let j = 0; j < arr.length; j++) {
      if (j !== i) {
        numStr += `(\\tilde{x}-x_${j})`
        denumStr += `(x_${i}-x_${j})`
      }
    }
    numerator.push(numStr)
    denum.push(denumStr)
  }
  return {
    numerator,
    denum
  }
}

export default {
  components: { LineChart },
  data () {
    return {
      /* eslint-disable */
      countedResultL : [],
      /* eslint-enable */
      fields: [],
      resultsL: [],
      answerL: [],
      answerLDec: [],
      grapg: '',
      renderChart: false,
      point: -7.86,
      diff: 0.3,
      start: -1,
      pointsAmmount: 4,
      resultWithNumbers: [],
      resultWithLetters: [],
      resultForEachArr: [],
      functionRes: '',
      functionNumbersResult: '',
      functionNumbersResultDec: ''
    }
  },
  methods: {
    init () {
      this.fields = []
      for (let i = 0; i < this.pointsAmmount; i++) {
        this.fields.push({
          labelX: 'X',
          valueX: `${Math.round10((Number(this.start) + (this.diff * i)), -1)}`,
          labelY: 'Y',
          valueY: '' }
        )
      }
    },
    reset () {
      this.countedResultL = []
      this.fields = []
      this.resultsL = []
      this.answerL = []
      this.answerLDec = []
      this.grapg = ''
      this.renderChart = false
      this.point = -7.86
      this.diff = 0.3
      this.start = -1
      this.pointsAmmount = 4
      this.resultWithNumbers = []
      this.resultWithLetters = []
      this.resultForEachArr = []
      this.functionRes = ''
      this.functionNumbersResult = ''
      this.functionNumbersResultDec = ''
    },
    addField () {
      this.fields.push(
        { labelX: 'X', valueX: '', labelY: 'Y', valueY: '' }
      )
    },
    removeField () {
      this.fields.pop()
    },
    countResultsL () {
      this.resultsL = []
      this.countedResultL = []
      this.answerL = []
      this.grapg = ''
      this.renderChart = false
      for (let i = 0; i < this.fields.length; i++) {
        let numerator = []
        let denominator = []
        for (let j = 0; j < this.fields.length; j++) {
          if (j !== i) {
            if (this.fields[j].valueX < 0) {
              numerator.push(`(x + ${Math.abs(parseFloat(this.fields[j].valueX))})`)
              denominator.push(`(${parseFloat(this.fields[i].valueX)} + ${Math.abs(parseFloat(this.fields[j].valueX))})`)
            } else {
              numerator.push(`(x - ${parseFloat(this.fields[j].valueX)})`)
              denominator.push(`(${parseInt(this.fields[i].valueX)} - ${Math.abs(parseInt(this.fields[j].valueX))})`)
            }
          }
        }
        this.resultsL.push({ numerator, denominator })
      }

      this.resultsL.map(el => {
        let exprNumer = algebra.parse(el.numerator[0])
        let exprDenom = algebra.parse(el.denominator[0])
        for (let i = 1; i < el.numerator.length; i++) {
          exprNumer = exprNumer.multiply(algebra.parse(el.numerator[i]))
          exprDenom = exprDenom.multiply(algebra.parse(el.denominator[i]))
        }
        this.countedResultL.push({
          'numerator': exprNumer.toString(),
          'denominator': exprDenom.toString()
        })
      })

      const { numerator: num1, denum: denum1 } = makeLetterString(this.resultsL)
      this.resultWithNumbers = num1
      this.resultWithLetters = denum1

      for (let i = 0; i < this.countedResultL.length; i++) {
        const expr = algebra.parse(`(${this.fields[i].valueY} * 1 / (${this.countedResultL[i].denominator}))(${this.countedResultL[i].numerator})`)
        this.answerL.push(expr.toString())
      }

      let graphExp = algebra.parse(this.answerL[0])
      for (let i = 1; i < this.answerL.length; i++) {
        graphExp = graphExp.add(algebra.parse(this.answerL[i]))
      }

      this.grapg = graphExp.toString()

      this.renderChart = true
      this.resultForEachArr = countWithPoint(this.resultsL, this.point)
      this.functionRes = countFunctionInPoint(this.fields, this.resultForEachArr)
      const lelel = algebra.parse(this.grapg)
      // Error here
      this.functionNumbersResult = lelel.eval({ x: getResultDecNumber(this.point) }).toString()
      this.functionNumbersResultDec = evaluate(this.functionNumbersResult)
    },
    paintChart () {
      this.renderChart(this.answerL)
    }
  },
  computed: {
    numeratorIndex1 () {
      return `$$ ${this.countedResultL[0].numerator} $$`
    },
    checkFields () {
      if (this.fields.length === 0) return false

      const showArray = this.fields.map(el => {
        return Object.values(el)
      })

      return showArray.flat().every(el => el !== '')
    }
  }
}
</script>
