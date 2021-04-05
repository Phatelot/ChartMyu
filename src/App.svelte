<script>
  import Base from "./Base.svelte";
  import { characters } from "./data";

  import { BMI, BMIToCategory, toLbs } from "./weight";

  const possibleValuesToPlot = ["kg", "lbs", "BMI"];
  $: valueToPlot = possibleValuesToPlot[0];

  const characterNames = Object.entries(characters).map(([name]) => name);
  const characterColors = Object.fromEntries(
    characterNames.map((name) => [name, characters[name].color])
  );

  let selectedCharacters = [...characterNames];

  $: dataFromSelectedCharacters = Object.entries(characters).filter(([name]) =>
    selectedCharacters.includes(name)
  );

  $: selectedWeighings = dataFromSelectedCharacters.map(([name, data]) => ({
    name,
    ...data,
    weighings: Object.entries(data.weighingsByDay).map(([day, weighing]) => ({
      day,
      weighing,
    })),
  }));

  $: getWeighingInfo = (datasetIndex, index) => {
    return {
      character: selectedWeighings[datasetIndex],
      ...selectedWeighings[datasetIndex].weighings[index],
    };
  };

  const maxDay = (() => {
    let maxDay = 0;

    for (const characterName in characters) {
      for (const day in characters[characterName].weighingsByDay) {
        const parsedDay = parseInt(day);
        if (parsedDay > maxDay) {
          maxDay = parsedDay;
        }
      }
    }

    return maxDay;
  })();

  const isTouchDevice =
    "ontouchstart" in window ||
    navigator.maxTouchPoints > 0 ||
    navigator.msMaxTouchPoints > 0;

  function toDataset(data, valueFunc) {
    return data.map(([name, data]) => ({
      label: name,
      borderColor: data.color,
      pointBackgroundColor: data.color,
      pointHoverBackgroundColor: data.color,
      pointBorderWidth: isTouchDevice ? 12 : 4,
      pointHoverBorderWidth: isTouchDevice ? 6 : 2,
      pointStyle: "rect",
      borderColor: data.color,
      fill: false,
      data: Object.entries(data.weighingsByDay).map(([day, weighing]) => ({
        x: day,
        y: valueFunc({
          height: data.height,
          weight: weighing.weight,
        }),
      })),
    }));
  }

  $: allDataLines = {
    kg: {
      datasets: toDataset(dataFromSelectedCharacters, ({ weight }) => weight),
    },
    lbs: {
      datasets: toDataset(dataFromSelectedCharacters, ({ weight }) =>
        toLbs(weight)
      ),
    },
    BMI: {
      datasets: toDataset(dataFromSelectedCharacters, ({ weight, height }) =>
        BMI(height, weight)
      ),
    },
  };

  $: dataLine = allDataLines[valueToPlot];

  $: options = {
    legend: {
      display: false,
    },
    tooltips: {
      enabled: false,
    },
    animation: false,
    scales: {
      xAxes: [
        {
          type: "linear",
          bounds: "ticks",
          ticks: {
            max: maxDay + 1,
            min: 0,
            stepSize: 5,
          },
        },
      ],
      yAxes: [
        {
          type: "linear",
          bounds: "ticks",
          ticks: {
            suggestedMax: {
              kg: 100,
              lbs: toLbs(100),
              BMI: 50,
            }[valueToPlot],
            min: 0,
            stepSize: 5,
          },
        },
      ],
    },
  };

  $: lastSelected = null;

  const toText = (lastSelected, valueToPlot) => {
    if (lastSelected === null) {
      return "";
    }

    let previousWeighingsFromLastSelected = Object.entries(
      lastSelected.character.weighingsByDay
    )
      .filter(([day]) => parseInt(day) < lastSelected.day)
      .filter(([_, weighing]) => !!weighing.weight)
      .sort(([day1], [day2]) => parseInt(day2) - parseInt(day1))[0];

    if (valueToPlot === "kg") {
      let text = `On day ${lastSelected.day}, ${lastSelected.character.name} weighs ${lastSelected.weighing.weight} kg.`;
      if (!!previousWeighingsFromLastSelected) {
        const weightDifference =
          Math.round(
            (lastSelected.weighing.weight -
              previousWeighingsFromLastSelected[1].weight) *
              10
          ) / 10;
        text += ` She gained ${weightDifference} kg in the last ${
          lastSelected.day - previousWeighingsFromLastSelected[0]
        } day(s).`;
      }
      return text;
    }
    if (valueToPlot === "lbs") {
      let text = `On day ${lastSelected.day}, ${
        lastSelected.character.name
      } weighs ${toLbs(lastSelected.weighing.weight)} lbs.`;
      if (!!previousWeighingsFromLastSelected) {
        const weightDifference =
          Math.round(
            toLbs(
              lastSelected.weighing.weight -
                previousWeighingsFromLastSelected[1].weight
            ) * 10
          ) / 10;
        text += ` She gained ${weightDifference} lbs in the last ${
          lastSelected.day - previousWeighingsFromLastSelected[0]
        } day(s).`;
      }
      return text;
    }
    if (valueToPlot === "BMI") {
      const lastBMI = BMI(
        lastSelected.character.height,
        lastSelected.weighing.weight
      );
      let text = `On day ${lastSelected.day}, ${
        lastSelected.character.name
      } has a BMI of ${lastBMI}, so she is ${BMIToCategory(lastBMI)}.`;
      if (!!previousWeighingsFromLastSelected) {
        const previousBMI = BMI(
          lastSelected.character.height,
          previousWeighingsFromLastSelected[1].weight
        );
        if (previousBMI !== lastBMI) {
          text += ` She gained ${lastBMI - previousBMI} BMI point in the last ${
            lastSelected.day - previousWeighingsFromLastSelected[0]
          } day(s).`;
        }
      }
      return text;
    }
  };

  const toPageUrl = (lastSelected) => {
    if (!lastSelected) {
      return "";
    }
    return lastSelected.weighing.url;
  };

  let clickPointHandler = (event) => {
    if (!Array.isArray(event)) {
      return;
    } else if (event.length == 0) {
      return;
    }
    [event] = event;
    lastSelected = getWeighingInfo(event._datasetIndex, event._index);
  };
</script>

<main>
  <div class="cm-container">
    <h1>Chart Myu ({valueToPlot})</h1>
    <div class="cm-chart">
      <Base data={dataLine} {options} {clickPointHandler} />
    </div>

    <form class="cm-select-char">
      {#each characterNames as characterName}
        <label
          class="cm-char-label"
          style="background-color: {characterColors[characterName]}"
        >
          <input
            class="cm-char-checkbox"
            type="checkbox"
            bind:group={selectedCharacters}
            value={characterName}
          />
          {characterName}
        </label>
      {/each}
    </form>

    <form class="cm-select-value">
      {#each possibleValuesToPlot as possibleValueToPlot}
        <label class="cm-value-label">
          <input
            class="cm-value-radio"
            type="radio"
            bind:group={valueToPlot}
            value={possibleValueToPlot}
          />
          {possibleValueToPlot}
        </label>
      {/each}
    </form>

    {toText(lastSelected, valueToPlot)}
    {#if toPageUrl(lastSelected)}
      <p>
        <a href={toPageUrl(lastSelected)}>Go to page</a>
      </p>
    {/if}
  </div>

  <div class="cm-myus-head">
    <img src="./myu.png" alt="myu's head" />
  </div>
  <footer>
    <p>All characters belong to Pixiveo.</p>

    <p>
      <a href="https://www.patreon.com/pixiveo">Support Pixiveo on Patreon</a>
    </p>

    <p>
      <a href="https://www.deviantart.com/pixiveo"
        >Pixiveo's Deviantart gallery</a
      >
    </p>

    <p>
      <a href="https://github.com/Phatelot/ChartMyu">Source code of this page</a
      >
    </p>
  </footer>
</main>

<style>
  h1 {
    text-align: center;
    color: rgb(65, 65, 65);
  }

  footer p {
    margin-left: 4px;
    margin-right: 4px;
    text-align: center;
  }

  img {
    max-width: 120px;
  }

  .cm-myus-head {
    display: flex;
    flex-flow: row-reverse;
  }

  .cm-container {
    padding: 18px;
    flex-grow: 1;
  }

  @media only screen and (min-width: 768px) and (max-width: 1000px) {
    .cm-container {
      padding-left: 100px;
      padding-right: 100px;
    }
  }

  @media only screen and (min-width: 1001px) and (max-width: 1300px) {
    .cm-container {
      padding-left: 200px;
      padding-right: 200px;
    }
  }

  @media only screen and (min-width: 1301px) {
    .cm-container {
      padding-left: 350px;
      padding-right: 350px;
    }
  }

  .cm-chart {
    background-color: white;
  }

  .cm-select-char {
    justify-content: center;
    display: flex;
    flex-wrap: wrap;
  }

  .cm-char-label {
    padding: 5px;
    border: none;
    border-radius: 5px;
    text-align: center;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 4px 2px;
  }

  .cm-select-value {
    justify-content: center;
    display: flex;
    flex-wrap: wrap;
  }

  .cm-char-checkbox {
    display: none;
  }

  .cm-value-label {
    padding: 5px;
    border-radius: 5px;
    border-width: 1px;
    text-align: center;
    border: solid;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    margin: 4px 2px;
  }

  .cm-value-radio {
    display: none;
  }
</style>