<script>
  import domtoimage from "dom-to-image";
  import html2canvas from "html2canvas";
  import { onMount } from "svelte";
  let redirect = "http://localhost:5000/"
  let client_id = "1461a32c547441d481b49799e368ff32";
  let client_secret = "7b9fbf1d4d724735993227e7602311d3";
  let access_token = localStorage.getItem("access_token");
  let refresh_token;
  let showReceipt = false;
  let songs;
  const auth = "https://accounts.spotify.com/authorize";
  const token = "https://accounts.spotify.com/api/token";
  const top = "https://api.spotify.com/v1/me/top/tracks?limit=10&time_range="

  function getAuthorization() {
    let url = auth;
    url += "?client_id=" + client_id;
    url += "&response_type=code";
    url += "&redirect_uri=" + encodeURI(redirect);
    url += "&show_dialog=true"
    url += "&scope=user-top-read"
    window.location.href = url;
  }
  
  function getCode() {
    return new URLSearchParams(window.location.search).get('code')
  }

  function fetchToken(code) {
    let data = "grant_type=authorization_code";
    data += "&code=" + code;
    data += "&redirect_uri=" + encodeURI(redirect);
    data += "&client+id=" + client_id;
    data += "&client_secret=" + client_secret;
    callAuthorizationApi(data);
  }

  function callAuthorizationApi(data) {
    let xhr = new XMLHttpRequest();
    xhr.open("POST", token, true);
    xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
    xhr.setRequestHeader("Authorization", "Basic " + btoa(client_id + ":" + client_secret));
    xhr.send(data);
    xhr.onload = handleAuthorizationResponse;
  }

  function handleAuthorizationResponse() {
    if (this.status == 200) {
      let res = JSON.parse(this.responseText);
      access_token = res.access_token;
      localStorage.setItem("access_token", access_token);
      refresh_token = res.refresh_token;
      localStorage.setItem("refresh_token", refresh_token);
    } else {
      console.log(this.responseText);
      window.alert(this.responseText)
    }
  }

  function callApi(access_token, length) {
    let url = top + length;
    let xhr = new XMLHttpRequest();
    xhr.open("GET", url, true);
    xhr.setRequestHeader("Content-Type", "application/json");
    xhr.setRequestHeader("Authorization", "Bearer " + access_token);
    xhr.send(null);
    xhr.onload = handleApiResponse;
  }

  function handleApiResponse() {
    if (this.status == 200) {
      console.log(JSON.parse(this.responseText))
      songs = JSON.parse(this.responseText).items;
      showReceipt = true;
    } else if (this.status == 401) {
      
    } else {
      console.log(this.responseText);
      window.alert(this.responseText)
    }
  }

  function getShortTerm() {
    callApi(access_token, "short_term");
  }

  function getMediumTerm() {
    callApi(access_token, "medium_term");
  }

  function getLongTerm() {
    callApi(access_token, "long_term");
  }


  function makeSongTag(index) {
    let title = songs?.[index].title
    let artists = songs?.[index].artists.map((artist, i) => artist.name)
    console.log({title, artists})
  }

  function handleRedirect() {
    const code = getCode();
    fetchToken(code);
    window.history.pushState("", "", redirect)
  }

  function getSongString(song) {
    return song?.name + " - " + song?.artists?.map((artist, i) => {
      if (i == 0) {
        return artist.name
      } else {
        return " " + artist.name
      }
    })
  }

  function getSongDuration(song) {
    const minutes = Math.floor(Math.ceil(song.duration_ms / 1000) / 60)
    let seconds = (Math.ceil(song.duration_ms / 1000) % 60)
    if (seconds <= 9) {
      seconds = "0" + seconds
    }
    return minutes + ":" + seconds
  }
  onMount(() => {
      if (window.location.search.length > 0) {
        showReceipt = true;
        handleRedirect();
      } else if (access_token == null) {
        console.log("No access token - should hide nutrition")
      } 
    });

    function consolelog() {
      console.log(songs)
      console.log(showReceipt)
    }

    function downloadImage() {
      var offscreen = document.querySelector('.nutrition-facts-container');
      html2canvas(offscreen).then((canvas) => {
          var dataUrl = canvas.toDataURL();
          var link = document.createElement('a');
          link.download = 'nutrition-label.jpeg';
          link.href = dataUrl;
          link.click();
      })
      /*
      domtoimage.toJpeg(document.getElementById("nutrition-label"), {quality: 0.95})
        .then(function (dataUrl) {
          var link = document.createElement('a');
          link.download = 'nutrition-label.jpeg';
          link.href = dataUrl;
          link.click();
        })
        ]*/
    }

</script>

<main style="display: flex;flex-direction: column;align-items: center;">
  <h1>Spotify Nutritional Facts</h1>
  <button on:click={getAuthorization}>{access_token ? "Logged in" : "Connect to Spotify"}</button>
  <div>
    <button on:click={getShortTerm}>Last Month</button>
    <button on:click={getMediumTerm}>Last 6 Months</button>
    <button on:click={getLongTerm}>All Time</button>
  </div>
  {#if showReceipt}
  <body>
    <div style="
    display: flex;flex-direction: column;align-items: center;">
    <div id="nutrition-label" class="nutrition-facts-container">
      <span class="title">Nutrition Facts</span>
      <hr class="hr-light" />
      <span class="nutrition-servings">1 serving per container</span>
      <div class="nutrition-serving-size">
        <span>Serving size</span>
        <span>10 songs</span>
      </div>
      <hr class="hr-bold" />
      <span class="nutrition-amount-per-serving">Amount per serving</span>
      <div class="nutrition-calories">
        <span>Calories</span>
        <span>230</span>
      </div>
      <hr />
      <span class="nutrition-daily-value">% Daily Value<sup>*</sup></span>

      <table class="nutrition-information" cellspacing="0" cellpadding="0">
        {#each songs as song, i}
        <tbody>
          <tr>
            <td>
              <span>{getSongString(song)}</span>
              <span>{i + 1}g</span>
            </td>
            <td>{getSongDuration(song)}</td>
          </tr>
        </tbody>
        {/each}
      </table>

      <hr />
      <div class="nutrition-disclaimer">
        <span>*</span>
        <span
          >The % Daily Value (DV) tells you how much a nutrient in a serving of
          food contributes to a daily diet. 2,000 calories a day is used for
          general nutrition advice.</span
        >
      </div>
    </div>
    <div>
    <button on:click={downloadImage}>Download Image</button>
  </div>
  </div>
  </body> 
  {/if}
</main>

<style>
  body {
    margin: 0;
  }

  .nutrition-facts-container {
    color: #000000;
    font-size: 1vw;
    width: 50%;
    display: inline-block;
    font-family: "Source Sans Pro", sans-serif;
    border: solid currentColor 0.8em;
    box-sizing: border-box;
    margin: 0;
    padding: 2em;
  }
  .nutrition-facts-container .title {
    display: block;
    font-size: 5.9em;
    font-weight: 900;
    line-height: 0.8em;
  }
  .nutrition-facts-container .nutrition-servings {
    font-size: 3em;
    line-height: 1.1em;
  }
  .nutrition-facts-container .nutrition-serving-size {
    display: table;
    width: 100%;
    font-weight: 900;
    font-size: 3.5em;
    line-height: 1.1em;
  }
  .nutrition-facts-container .nutrition-serving-size span {
    display: table-cell;
  }
  .nutrition-facts-container .nutrition-serving-size span:last-child {
    text-align: right;
  }
  .nutrition-facts-container .nutrition-amount-per-serving {
    font-weight: 700;
    font-size: 3em;
  }
  .nutrition-facts-container .nutrition-calories {
    display: table;
    width: 100%;
    font-weight: 900;
    font-size: 5em;
    line-height: 0.5em;
    margin-bottom: 0.2em;
  }
  .nutrition-facts-container .nutrition-calories span {
    display: table-cell;
  }
  .nutrition-facts-container .nutrition-calories span:last-child {
    text-align: right;
    font-size: 1.4em;
  }
  .nutrition-facts-container .nutrition-daily-value {
    font-size: 3.5em;
    text-align: right;
    display: block;
    font-weight: 700;
  }
  .nutrition-facts-container .nutrition-information {
    width: 100%;
    font-size: 2em;
    border-collapse: collapse;
  }
  .nutrition-facts-container .nutrition-information.nutrition-vitamins td,
  .nutrition-facts-container .nutrition-information.nutrition-vitamins span {
    font-weight: 400 !important;
  }
  .nutrition-facts-container .nutrition-information tbody tr td {
    border-top: 0.2em solid currentColor;
    padding: 2px;
  }
  .nutrition-facts-container .nutrition-information tbody tr td:last-child {
    text-align: right;
    font-weight: 700;
  }
  .nutrition-facts-container
    .nutrition-information
    tbody
    tr:first-child
    span:first-child {
    font-weight: 700;
  }
  .nutrition-facts-container
    .nutrition-information
    tbody
    tr:not(:first-child)
    td {
    padding-left: 1em;
  }
  .nutrition-facts-container .nutrition-disclaimer {
    font-size: 2.2em;
    display: table;
    padding-top: 0.4em;
  }
  .nutrition-facts-container .nutrition-disclaimer span:last-child {
    display: table-cell;
  }
  .nutrition-facts-container hr {
    display: block;
    border: none;
    border-bottom: solid currentColor 2em;
  }
  .nutrition-facts-container .hr-light {
    border-width: 0.5em;
  }
  .nutrition-facts-container .hr-bold {
    border-width: 2em;
  }
  .nutrition-facts-container .indent {
    padding-left: 2em !important;
  }
  .nutrition-facts-container .bold {
    font-weight: 700;
  }

  .some-other-content {
    display: inline-block;
  }
</style>
