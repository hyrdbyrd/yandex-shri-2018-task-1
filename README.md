# Для запуска
Для начала, можно запустить eslint
```
eslint .
```
Или
```
npm run lint
```
Для запуска самой странички нужно ввести:
```
npm start
```
# Ошибки, из-за которых карта, не отображалась
## Import
Браузер сообщил об ошибке - TypeError.
С помощью дебагера, узнал, что при импорте, в initMap уходит объект.
Исправить ошибку можно было двумя способами:
#### 1. src/map
```js
// ...
export default function initMap(ymaps, containerId) {
// ...
```
#### 2. src/index
```js
import { initMap } from './map';
// ...
```
Так как, во всех файлах, используется второй вариант, решил второй вариант.
Оба варианты, дадут одинаковый эффект.
## CSS
Ошибка пропала, однако карта всё еще не отображалась.
Заглянул в элемент, увидел, что карта элемент изменился, и в нём находятся ymaps элементы.
Пришел к выводу, что дело в отображении css - размеры.
#### public/index.css
```css
html,
body,
#map {
    margin: 0;
    height: 100vh;
}
```
# Ошибки, из-за которых, не отображались метки
## ObjectManager
После загрузки карты, не отображались метки.
Проведя некоторое время в документации, и посмотрев на код, пришел к выводу, что они нигде не выводятся.
Найдя способ добавления менеджера объектов в карту, использовал нужное действие
#### src/map
```js
// ...
myMap.geoObjects.add(objectManager);
// ...
```
# Ошибки, из-за которых метки, отображались не в соответствии с условиями
## LAT, LONG
Метки стали отображаться, однако, вовсе не на Москве.
Побродил по коду, обнаружил где задаються координаты.
Решил проблему, переставив длину, с шириной.
#### src/mappers
```js
// ...
return {
    type: 'FeatureCollection',
    features: serverData.map((obj, index) => ({
      id: index,
      type: 'Feature',
      isActive: obj.isActive,
      geometry: {
        type: 'Point',
        coordinates: [obj.lat, obj.long] // [long, lat] = [lat, long]
      },
      properties: {
        iconCaption: obj.serialNumber
      },
      options: {
        preset: getObjectPreset(obj)
      }
    }))
  };
// ...
```
# Ошибки, из-за которых, не открывался балун (попап)
## Стрелочные функции
При клике на любую метку, выходила ошибка.
Увидев что в details.js находяться стрелочные функции у методов, которые использую this, немедленно поменял их в вид function () {}, т.к. у стрелочных функций, отсутствует собственный контекст.
#### src/details.js
```js
// ...
{
  build: function () { // () => {} ---> function() {}
    BalloonContentLayout.superclass.build.call(this);

    const { details } = this.getData().object.properties;
    if (details) {
      const container = this.getElement().querySelector('.details-chart');

      this.connectionChart = createChart(
        container,
        details.chart,
        details.isActive
      );
    }
  },
  clear: function () { // () => {} ---> function() {}
    if (this.connectionChart) {
      this.connectionChart.destroy();
    }

    BalloonContentLayout.superclass.clear.call(this);
  }
}
// ...
```
# Ошибки, из-за которых контент балуна, не соответствовал нужному
## Chart.js
Балуны открывалсь, однако не весь контент отабражался правильно.
График canvas графика присутствовал, однако сам график не отображался.
Пройдя по нужно коду, обнаружил, что для оси ординат, максимальным значением, является 0.
Посмотрев значения details.chart, понял что все значения являются положительными, следовательно то свойство, и было ошибкой.
```js
// ...
const chart = new Chart(ctx, {
    type: 'line',
    data: {
      labels: data.map(getLabel),
      datasets: [
        {
          data: data,
          borderWidth: 1,
          borderColor: borderColor,
          backgroundColor: backgroundColor
        }
      ]
    },
    options: {
        legend: { 
            display: false
        },
        scales: {
            xAxes: [{ ticks: { display: false } }],
            yAxes: [{ ticks: { beginAtZero: true } }] // delete yAxes[0].max;
        }
    }
  });
// ...
```
# Ошибки, из-за которых кластер, отображался не в соответствии с условиями
## objectManager.ClusterCollection
Так как, пройдясь по всему коду, я не обнаружил нужный фрагмент, понадобилась написать собственный код.
Не долго рыская в интернете, обнаружил простое решение, для того чтобы пройтись по всем объектам, внутри каждого кластера.
#### src/map.js
```js
// ...
// clusters
objectManager.clusters.events.add('add', event => {
    const cluster = event.get('child');
    const geoObjects = cluster.properties.geoObjects;
    let preset;

    if (geoObjects.every(obj => !obj.isActive)) {
      preset = 'red';
    } else if (geoObjects.some(obj => !obj.isActive)) {
      preset = 'yellow';
    }

    preset && objectManager.clusters.setClusterOptions(cluster.id, {
      preset: `islands#${preset}ClusterIcons`
    });
  });
// ...
```
# Перечень стилистических ошибок
* #### src/mappers.js
    ```js
    // before
    geometry:
    {
        type: "",
        coordinates: [] 
    }
    
    // after
    geometry: {
        type: '',
        coordinates: []
    }
    ```
* #### src/chart.js
    ```js
    // before
    datasets: [
        {
          data: data,
          borderWidth: 1,
            borderColor: borderColor,
                backgroundColor: backgroundColor
        }
    ]
    
    // after
    datasets: [
        {
          data: data,
          borderWidth: 1,
          borderColor: borderColor,
          backgroundColor: backgroundColor
        }
    ]
    ```
* #### src/details.js
    ```js
    // before
    {% if (properties.details) %}
        <div class="details-info">
            <div class="details-label">base station</div>
            <div class="details-title">{{properties.details.serialNumber}}</div>
            {% if (properties.details.isActive) %}
            <div class="details-state details-state_active">active</div>
            {% else %}
            <div class="details-state details-state_defective">defective</div>
            {% endif %}
            <div class="details-state details-state_connections">
                connections: {{properties.details.connections}}
            </div>
        </div>
        <div class="details-info">
            <div class="details-label">connections</div>
            <canvas class="details-chart" width="270" height="100" />
        </div>
    {% else %}
        <div class="details-info">
            Идет загрузка данных...
        </div>
    {% endif %}
    
    // after 
    {% if (properties.details) %}
      <div class="details-info">
        <div class="details-label">
          base station
        </div>
        <div class="details-title">
          {{properties.details.serialNumber}}
        </div>
        {% if (properties.details.isActive) %}
        <div class="details-state details-state_active">
          active
        </div>
        {% else %}
        <div class="details-state details-state_defective">
          defective
        </div>
        {% endif %}
        <div class="details-state details-state_connections">
            connections: {{properties.details.connections}}
        </div>
      </div>
      <div class="details-info">
        <div class="details-label">
          connections
        </div>
        <canvas class="details-chart" width="270" height="100"/>
      </div>
      {% else %}
        <div class="details-info">
            Идет загрузка данных...
        </div>
      {% endif %}
    ```
* #### src/api.js
    ```js
    // before
    export function loadDetails(id)
    {
      return fetch(`/api/stations/${id}`).then(response => response.json());
    }
    
    // after
    export function loadDetails(id) {
      return fetch(`/api/stations/${id}`)
        .then(response => response.json());
    }
