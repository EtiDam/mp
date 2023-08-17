<script>
  // CORE IMPORTS
  import { setContext, onMount } from "svelte";
  import axios from 'axios';  // Ajout du module axios
  import { getMotion } from "./utils.js";
  import { themes } from "./config.js";
  import ONSHeader from "./layout/ONSHeader.svelte";
  import ONSFooter from "./layout/ONSFooter.svelte";
  import Header from "./layout/Header.svelte";
  import Section from "./layout/Section.svelte";
  import Media from "./layout/Media.svelte";
  import Scroller from "./layout/Scroller.svelte";
  import Filler from "./layout/Filler.svelte";
  import Divider from "./layout/Divider.svelte";
  import Toggle from "./ui/Toggle.svelte";
  import Arrow from "./ui/Arrow.svelte";
  import Arrow2 from "./ui/Arrow2.svelte";
  import Em from "./ui/Em.svelte";
  import * as d3 from "d3"
  import { feature } from 'topojson-client';
  import Legende from './layout/Legende.svelte';

  // DEMO-SPECIFIC IMPORTS
  import bbox from "@turf/bbox";
  import { getData, setColors, getTopo, getBreaks, getColor } from "./utils.js";
  import { colors, units } from "./config.js";
  import { ScatterChart, LineChart, BarChart } from "@onsvisual/svelte-charts";
  import { Map, MapSource, MapLayer, MapTooltip } from "@onsvisual/svelte-maps";
    import { each } from "svelte/internal";

  let theme = "light";
  setContext("theme", theme);
  setColors(colors, theme);

  let data = {};
  let metadata = {};
  let metadataLoaded = false;
  let geojson;
  let map = null;
  let mapKey = "density";
  let selected; 
  let animation = getMotion();
  let hovered;
  let mapHighlighted = [];
  let isMapLoaded = false;

  const threshold = 0.65;

  let id = {}; // Object to hold visible section IDs of Scroller components
	let idPrev = {}; // Object to keep track of previous IDs, to compare for changes
	onMount(() => {
		idPrev = {...id};
	});

  let housePriceData = [];
  let incomeData = [];
  let ratioData = [];

  const topojson = "./data/communes-belges0 copy.json";

  const mapstyle = "https://api.maptiler.com/maps/7a398d58-c7b0-4b18-86df-50b59985bef1/style.json?key=KxWuaRoNgdIOk76i21Fs";
  const mapbounds = {
    belgium: [
      [2.5, 49.5], // coin inférieur gauche : longitude, latitude
      [6.4, 51.5]  // coin supérieur droit : longitude, latitude
    ]
  };

  let initialCenter = [4.5, 50.5]; // Remplacer par les coordonnées du centre initial de votre carte
  let initialZoom = 7; // Remplacer par le niveau de zoom initial de votre carte

  // Sélecteur d'âge
  let ageRanges = ["Moins de 24 ans", "De 25 à 29 ans", "De 30 à 34 ans", "De 35 à 39 ans", "De 40 à 44 ans", "De 45 à 49 ans", "De 50 à 54 ans","De 55 à 59 ans","De 60 à 64 ans","De 65 à 69 ans","De 70 à 74 ans","De 75 à 79 ans","De 80 à 84 ans","85 ans et plus"];
  let selectedAgeIndex = ageRanges.indexOf("De 50 à 54 ans");
  let selectedAgeRange = "De 50 à 54 ans"; 
  let ageIncomeData = [];

  // Define color scale
  let colorScale;

  function doSelect(e) {
    console.log(e);
    selected = e.detail.id;
    if (e.detail.feature) fitById(selected);
  }

  function doHover(e) {
    hovered = e.detail.id;
    let tooltipContent = '';
    if (hovered && metadata.lookup[hovered]) {
      tooltipContent = `${metadata.lookup[hovered].name} 
      - Prix médian pour un appartement: ${metadata.lookup[hovered].ageData["prix médian pour un appartement"]} 
      - Prix médian pour une maison: ${metadata.lookup[hovered].ageData["prix médian pour une maison"]}`
    }
  }


  function handleDblClick(e) {
    map.flyTo({center: e.lngLat, zoom: map.getZoom() - 1});
  }

  function resetView() {
    map.flyTo({center: initialCenter, zoom: initialZoom});
  }

  function fitBounds(bounds) {
    if (map) {
      map.fitBounds(bounds, {animate: animation, padding: 30});
    }
  }

  function fitById(id) {
    if (geojson && id) {
      let feature = geojson.features.find(d => d.properties.ID_4 == id);
      if (feature) {
        let bounds = bbox(feature.geometry);
        fitBounds(bounds);
      }
    }
  }

  function zoomToCoordinates(coordinates, zoomLevel) {
      if (map) {
          map.flyTo({ center: coordinates, zoom: zoomLevel });
      }
  }

  const actions = {
		map: { // Actions for <Scroller/> with id="map"
			map01: () => { // Action for <section/> with data-id="map01"
      resetView();
			},
			map02: () => {
      // Remplacez [longitude, latitude] par les coordonnées exactes de votre commune
      zoomToCoordinates([4.485, 50.730], 8);

      },

      map03: () => {
            selectedAgeIndex = ageRanges.indexOf("De 25 à 29 ans");
            resetView();
            handleAgeSelect();
             // Ceci mettra à jour les couleurs et les données liées à la tranche d'âge
        },
      map04: () => {
          resetView();
          fitById(365);
          handleAgeSelect();
            // Ceci mettra à jour les couleurs et les données liées à la tranche d'âge
      }
    }
	};

  	// Code to run Scroller actions when new caption IDs come into view
	function runActions(codes = []) {
		codes.forEach(code => {
			if (id[code] != idPrev[code]) {
				if (actions[code][id[code]]) {
					actions[code][id[code]]();
				}
				idPrev[code] = id[code];
			}
		});
	}
	$: id && runActions(Object.keys(actions)); // Run above code when 'id' object changes

  function handleZoomOut() {
    map.flyTo({center: map.getCenter(), zoom: map.getZoom() - 1});
  }
    

  function loadSourceAndThen(callback) {
      if (map.getSource('commune')) {
          callback();
      } else {
          map.once('sourcedata', () => {
              if (map.isSourceLoaded('commune')) {
                  callback();
              } else {
                  loadSourceAndThen(callback);
              }
          });
      }
  }

  function updateColors() {
      if (map && metadata) {
          loadSourceAndThen(() => {
              metadata.array.forEach(feature => {
                  if (feature.ageData[selectedAgeRange]) {
                      let value = Number(feature.ageData[selectedAgeRange]);
                      let colorValue = colorScale(value);
                      map.setFeatureState({source: 'commune', id: feature.code}, { color: colorValue });
                  }
              });
          });
      }
  }

  function handleAgeSelect() {
    selectedAgeRange = ageRanges[selectedAgeIndex];
    updateColors();

    // Trouver les revenus associés à la tranche d'âge sélectionnée
    const selectedIncome = ageIncomeData.find(d => d.age === selectedAgeRange);
    
    if (selectedIncome) {
      // Mettre à jour l'affichage (ou faire tout autre traitement nécessaire)
      console.log(`Revenus par an pour ${selectedAgeRange}: ${selectedIncome.annualIncome}`);
    }
  }


  function transformAgeRangeFormat(ageRange) {
    if (ageRange.startsWith("De")) {
      return "entre" + ageRange.slice(2).replace("à", "et");
    } else if (ageRange.startsWith("Moins")) {
      return ageRange.charAt(0).toLowerCase() + ageRange.slice(1);
    } else if (ageRange === "85 ans et plus") {
      return "plus de 85 ans";
    }
    return ageRange;
  }




  let ageRangesForSentences = ageRanges.map(transformAgeRangeFormat);

  function exampleFunction() {
    console.log(`J'ai choisi la tranche ${ageRangesForSentences[selectedAgeIndex]}.`);
  }



  onMount(async () => {
    // Fetch all necessary data
    let geo = await getTopo(topojson, 'Communes');

    geojson = geo; 
    geo.features.sort((a, b) => a.properties.NAME_4.localeCompare(b.properties.NAME_4));

    // Collect age data
    let ageData = geo.features.map(d => {
      let ageProps = {};

      // Loop through all properties of the feature
      for (let propName in d.properties) {
        // If the property starts with "De", "Moins", or "85 ans et plus", then it's an age range
        // Also add "prix médian pour une maison" and "prix médian pour un appartement" properties
        if (propName.startsWith("De") || propName.startsWith("Moins") || propName.startsWith("85 ans et plus") || propName === "prix médian pour une maison" || propName === "prix médian pour un appartement") {
          // Add the data to the ageProps object
          ageProps[propName] = d.properties[propName];
        }
      }

      return {
        code: d.properties.ID_4,
        name: d.properties.NAME_4,
        parent: d.properties.ID_3 ? d.properties.ID_3 : null,
        ageData: ageProps // Add the age data here
      };
    });

    const incomeResponse = await axios.get('./data/Revenus moyens par age.csv');
    const csvData = d3.csvParse(incomeResponse.data);
    
    ageIncomeData = csvData.map(row => {
      return {
        age: row['âge'],
        annualIncome: parseFloat(row['revenus par an'].replace(/,/g, '.')),
        monthlyIncome: parseFloat(row['revenus par mois'].replace(/,/g, '.').replace(/ /g, '')),
        loanLowEnd: parseFloat(row['emprunt sur trente ans (bas)'].replace(/ /g, '').replace(/€/g, '').replace(/,/g, '.').trim()),
        loanHighEnd: parseFloat(row['emprunt sur trente ans (haut)'].replace(/ /g, '').replace(/€/g, '').replace(/,/g, '.').trim())
      };
    });


    let lookup = {};
    ageData.forEach(d => {
      lookup[d.code] = d;
    });
    metadata.array = ageData;
    metadata.lookup = lookup;

    // Define color scale
    let colorThresholds = [0.05,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9, 1,1.1,1.2,1.3,1.4,1.5,1.6,1.7,1.8,2, 3];
    let colorRange =['#fef8f8', '#fad7e5', '#f1b6d3', '#e597c1', '#d479b0', '#c05ca1', '#a84192', '#8d2783', '#6e1076', '#49006a','#49006a', '#4e267b', '#543f8b', '#5b569a', '#636ca9', '#6d82b7', '#7997c5', '#88add2', '#98c3de', '#abd9e9']

    colorScale = d3.scaleThreshold()
      .domain(colorThresholds)
      .range(colorRange); 

  });

  handleAgeSelect();

</script>






<ONSHeader filled={true} center={false} />

<Header bgimage="./img/Projet.jpg" bgfixed={true} theme="light" short={true} center={false}>
  <h1><Em color="white" >Les jeunes et l’immobilier :<br /> quand le rêve tourne au mirage</Em></h1>
  <p class="text-big" style="margin-top: 25px">
    <Em color="white" >Face à un marché de plus en plus austère,<br />
      la jeunesse voit son rêve de propriété  s'effriter<br />
     et un écart se creuser entre les générations.
    </Em>
  </p>
  <p style="margin-top: 20px">
   <Em color= "white"> 16/08/2023</Em>
  </p>
  <p>
    <Em color= "white"><Toggle label="Animation {animation ? 'on' : 'off'}" mono={true} bind:checked={animation} /></Em>
  </p>
  <div style="text-align: center;">
    <Arrow color="white" {animation} />
  </div>
</Header>

<Divider></Divider>

<Section theme="light">
  <p class="mb">
    Devenir propriétaire de son logement : certains jeunes en rêvent, les autres ne se font plus d’illusion.  Et ce n’est pas le nouveau baromètre de la Fédération du Notariat (Fednot) qui devrait les rassurer. En 2023 <Em color="black">les acheteurs de moins de 30 ans</Em> ne représentaient que 27% des transactions, contre 30% en 2018. Selon la Banque nationale de Belgique (BNB), l'âge moyen de l'achat d’un premier logement est lui en constante augmentation depuis plusieurs années. Il est passé de 30 à 36 ans depuis la fin des années 1990. 
  </p>  
  <p class="mb">
    La baisse du nombre d'acheteurs jeunes et la hausse de l'âge du premier achat touchent toutes les régions et illustrent la réalité du marché immobilier belge, celle d’un accès à la propriété qui se déteriore pour les moins de 30 ans. 
  </p>
  <p class="mb">
    Aujourd'hui plus qu'hier, l'accès au marché immobilier connait, en plus des inégalités sociales, l'importance grandissante des inégalités intergénérationelles. Car on ne choisit pas plus sa famille que la période économique qui nous accueille.
  </p>

  <p class="mb">  
    Pour observer cette disparité et comprendre son évolution, nous allons devoir mesurer cette accessibilité au logement. L'approche classique, utilisée notamment par la Banque nationale de Belgique (BNB), consiste à comparer l’évolution des revenus des ménages à celle des prix des logements.
  </p>

</Section>


<Filler theme="lightblue" short={true} wide={true} center={false}>
  <p class="text-big">
    Pour mieux comprendre la situation, sélectionnez votre catégorie d’âge.
    Cela va nous permettre de visualiser les différences entre les générations et les conséquences sur leur accès au logement. 
  </p>
  <h2> Quel âge avez-vous?</h2>


 <p> <select class="tooltipdecodeurs" bind:value="{selectedAgeIndex}" on:change="{handleAgeSelect}">
    {#each ageRanges as ageRange, index}
      <option value={index}>{ageRange}</option>
    {/each}<Em color="black"></Em>
  </select></p>
</Filler>

<Filler theme="light" short={true} wide={true} center={false}>
  <p class="mb">
    Vous avez <Em color="black">{ageRangesForSentences[selectedAgeIndex]}</Em>? En 2020, les Belges de cette tranche d'âge avaient un revenu annuel moyen de <Em color="black">{ageIncomeData[selectedAgeIndex] ? ageIncomeData[selectedAgeIndex].annualIncome : ''} €</Em>.
    Cette somme va nous servir de référence pour observer l'influence de l'âge sur l'accès à la propriété dans notre pays.  
  </p>
</Filler>


{#if geojson}
<Scroller {threshold} bind:id={id['map']}>
  <div slot="background">
    <figure>
      <div class="col-full height-full">
        <Map on:load={updateColors} style={mapstyle} bind:map={map} interactive={false} location={{bounds: mapbounds.belgium}}>
          <MapSource id="commune" type="geojson" data={geojson} promoteId="ID_4" maxzoom={13}>
              <MapLayer id="commune-fill" idKey="ID_4" data={geojson.features} type="fill"
                        select {selected} on:select={doSelect} 
                        hover {hovered} on:hover={doHover} 
                        highlight highlighted={mapHighlighted} 
                        paint={{
                          'fill-color': ['case', ['!=', ['feature-state', 'color'], null], ['feature-state', 'color'], 'rgba(0, 0, 0, 0.1)'],
                          'fill-opacity': 0.8
                        }}>
                 <MapTooltip content={
                  hovered && metadata.lookup[hovered]
                      ? `<div class="tooltipdecodeurs">
                          <div class="titre_tt">${metadata.lookup[hovered].name}</div>
                          <div class="hr" style="
                          border-top: 1px solid #e8eaee;"></div>
                          ${metadata.lookup[hovered].ageData["prix médian pour une maison"] && metadata.lookup[hovered].ageData["prix médian pour une maison"] !== "nan€" ? "<div>Prix médian pour une maison: " + metadata.lookup[hovered].ageData["prix médian pour une maison"] + "</div>" : ""}
                          ${metadata.lookup[hovered].ageData["prix médian pour un appartement"] && metadata.lookup[hovered].ageData["prix médian pour un appartement"] !== "nan€" ? "<div>Prix médian pour un appartement: " + metadata.lookup[hovered].ageData["prix médian pour un appartement"] + "</div>" : ""}
                         </div>`
                      : ''
                  } />
               
                            
                
                  
              </MapLayer>
              <MapLayer id="commune-line" type="line" 
                        paint={{
                          'line-color': ['case', ['==', ['feature-state', 'hovered'], true], 'orange', ['==', ['feature-state', 'selected'], true], 'black', ['==', ['feature-state', 'highlighted'], true], 'black', 'rgba(255,255,255,0)'],
                          'line-width': 2
                        }}/>
          </MapSource>

          <div id="controls-container" style="position: absolute; bottom: 10px; left: 10px; background-color: white; padding: 10px; border-radius: 5px; display: flex; flex-direction: column; align-items: center;">

            <!-- Première partie: Bouton Dézoomer -->
            <button class="dezoomer-button" on:click={resetView}>
                Dézoomer
            </button>
          
            <!-- Diviseur -->

            <!-- Diviseur -->
            <div style="height: 1px; background-color: grey; width: 80%; margin: 5px 0;"></div>
        
            <!-- Troisième partie: Informations d'abordabilité -->
            <div class="titre_tt" style="display: flex; justify-content: space-between; margin-top: 10px; width: 100%;">
                <span style="font-size: 1rem; color: #000;">Plus abordable</span>
                <span style="font-size: 1rem; color: #000;">Moins abordable</span>
            </div>
            
            <Legende />
        
          </div>
        
        
         
      </Map>
      </div>
    </figure>
  </div>


  <div slot="foreground">
    <section data-id="map01">
      <div class="col-medium">
        <p>
          Voici une carte qui représente les communes selon leur accessibilité. Les communes <Em color="blue">bleues</Em> sont les plus accessibles, à l'inverse les communes <Em color="pink">roses</Em> sont trop chères pour la moyenne des Belge ayant <Em color="elegant">{ageRangesForSentences[selectedAgeIndex]}</Em>.  
        </p>
      </div>
    </section>

    <section data-id="map01">
      <div class="col-medium">
        <p>
          Survolez la carte pour découvrir combien coûte une maison et/ou un appartement dans votre commune en 2023.
        </p>
      </div>
    </section>

    <section data-id="map01">
      <div class="col-medium">
        <p>
          En utilisant le revenu moyen de chaque tranche d'âge, nous pouvons estimer la somme que vous pourriez emprunter selon les critères actuels de prêts actuellement appliqués par les banques belges. Ce montant est ensuite comparé aux prix de l’immobilier dans chaque commune.
        </p>
      </div>
    </section>
    
    
    <section data-id="map01">
      <div class="col-medium">
            <p>
                En déplacant ce slider vous pourrez constater les différences entre les générations:
            </p>        
    
            <div class="custom-slider-container">
                <input type="range" style="width:657px;" min="0" max="{ageRanges.length - 1}" bind:value="{selectedAgeIndex}" on:change="{handleAgeSelect}" class="custom-slider">
            </div>
    
            <!-- Conteneur pour les informations de tranche d'âge et de revenu -->
            <div class="info-container" style="display: flex; justify-content: space-between; max-width: 80%; margin-left: auto; margin-right: auto;">
              <div class="info-box">
                <label for="ageRangeBox"><Em color="black">Tranche d'âge</Em></label>
                <div id="ageRangeBox" class="data-box">{ageRanges[selectedAgeIndex]}</div>
              </div>
              <div class="info-box">
                <label for="annualIncomeBox"><Em color="black">Revenu annuel</Em></label>
                <div id="annualIncomeBox" class="data-box">{ageIncomeData[selectedAgeIndex] ? ageIncomeData[selectedAgeIndex].annualIncome : ''} €</div>
              </div>
            </div> 
        </div>
    </section>
    
    

    <section data-id="map03">
      <div class="col-medium">
        <p>
          On le remarque assez vite, au sein de la population active, ce sont les Belges âgés de 25 à 29 ans qui sont les moins bien lotis. Voici à quoi ressemble leur carte. 
        </p>
      </div>
    </section>

      <section data-id="map02">
      <div class="col-medium">
        <p>
          C’est dans le Brabant Wallon que les acheteurs de moins de 30 ans sont les plus rares. Ils sont passés de <Em color="black">20,2%</Em> en 2018 à <Em>16,2%</Em> en 2023, estime Fednot. La faute au confinement de 2020 et à la flambée des prix des maisons avec jardins se situant proches de la capitale. 
        </p>
      </div>
    </section>

    <section data-id="map04">
      <div class="col-medium">
        <p>
          A l’inverse, le porte-parole de Fednot Renaud Grégoire, indique que certaines communes comme Courcelles, Sambreville, La Louvière ou Binche connaissent un taux de pénétrations plus élevé. 
        </p>
      </div>
    </section>

    <section data-id="map04">
      <div class="col-medium">
        <p>
          « <cite>Si les plus jeunes sont encore présents sur le marché immobilier, c'est parce qu'ils vont dans les communes les moins chères. Les moins de 30 ans posent <Em color="black">le prix</Em> comme critère principal. Ils iront donc chercher des biens moins accessibles à des tarifs plus avantageux</cite> », analyse Renaud Grégoire.
        </p>
      </div>
    </section>

   
  </div>



</Scroller>

{/if}

<Section>
  <p class="mb">
    Le revenu plus faible des moins de 30 ans joue donc un rôle majeur dans ces inégalités mais il n'explique pas, à lui seul, ce recul de l'accès à la propriété chez les jeunes. Si les écarts de revenus entre les classes d'âge ne sont pas neufs, deux éléments ont, eux, radicalement empiré la situation : <Em color="blue">le prix de l'immobilier</Em> et les critères requis pour <Em color="green">un emprunt hypothécaire</Em>.
 </p>


  <p class="mb">
    Dans une étude publiée en juillet 2022, la BNB posait en titre d'un rapport, la question suivante: «La propriété est-elle à la portée de tous en Belgique ?» La réponse de l'institution financière était pessimiste et désignait comme première responsable, la hausse continue du prix des logements ces 50 dernières années. 
 </p>

 <div class="flourish-embed flourish-chart" data-src="visualisation/14705524"><script src="https://public.flourish.studio/resources/embed.js"></script></div>

	<p>
    La BNB qui dispose de données fiables depuis près de 50 ans pose un constat clair: «<cite>Depuis 1973, les prix des logements en Belgique ont été multipliés par un facteur 15. En prenant en compte l’inflation, <strong>les prix réels ont triplé</strong></cite>» , révèle l'instution financière. Les prix réels sont ceux qui prennent en compte le revenu disponible pour acquérir un bien. Alors que depuis 1990, les prix des logements ont augmenté entre 120 et 180% selon le type de biens, les salaires n’ont augmenté que de 27 % sur la même période.</p>
	<p>
   Une augmentation sur le long terme donc, mais qui jusqu'à récemment, s'équilibrait avec des taux d'intérêts historiquement bas sur les emprunts hypothécaires. Ce sont ces faibles taux qui ont permis pendant longtemps à des ménages de s'endetter et d'accéder à la propriété. La BNB le rappelle : « <cite>une diminution des taux hypothécaires entraîne un allégement de la charge de remboursement des ménages pour un montant emprunté donné. Corollairement, pour une même charge de remboursement, le montant qui peut être emprunté augmente.</cite>» Mais depuis quelques mois, la machine s'est grippée.
	</p>

  <h2 class="section-title">La fin des taux attractifs?</h2>

	<p>
    Pour ceux qui suivent le sujet, même les moins attentifs, la nouvelle n’en est pas vraiment une. La pandémie de Covid-19 puis la guerre en Ukraine ont causé une forte inflation. Les banques centrales ont relevés leurs taux directeurs et ont fait flamber les taux hypothécaires par la même occasion. Chaque mois les chiffres battent de nouveaux records, atteignant des seuils que l’on avait pas vus depuis parfois 12 ans. 
  </p>

  <div class="flourish-embed flourish-chart" data-src="visualisation/13615522"><script src="https://public.flourish.studio/resources/embed.js"></script></div>
  
  <p>
    Les nombres de nouveaux crédits ont nettement diminué en un an, de l’ordre de 30% pour les crédits à la construction et jusqu’à 45% pour les crédits de rénovation. Les particuliers sont contraints de repousser, voire de tirer un trait sur leurs projets immobiliers. Ceux qui décident toutefois de s’y risquer voient leur demande d’apport augmenter de façon conséquente.	
  </p>

  


</Section>


<Scroller>
  <div slot="background" class="col-wide height-full">
    <figure style="background-image: url('./img/sara.jpg'); background-position: 15% center; background-repeat: no-repeat; background-size: 50% auto;">
      <div class="col-wide height-full"></div>
    </figure>
  </div>

  <div slot="foreground">
   
    <section>
      <div class="col-medium left-centered-text">
        <blockquote class="citation-emphase">
          «Un appartement à ce prix-là, ça n'existe pas!»
          <div class="attribution">- Sara, 27 ans</div>
        </blockquote>
      </div>
    </section>
  
    <section>
      <div class="col-medium left-centered-text">
        <p>
          «<cite>Les sommes exigées pour un prêt sont énormes, cela m’a choquée</cite>», confirme Sara, directrice des ventes dans une agence digitale à Waterloo. A 27 ans, elle et son conjoint viennent d’acquérir un appartement à 460 000 € après des semaines de démarches.
        </p>
        <p>
          « <cite>Depuis quelques années, je souhaitais devenir propriétaire. Je touche un salaire plus que correct et mon conjoint est patron de sa propre entreprise de jardinage. Avec l’arrivée de ma fille en 2022, il fallait que nous quittions notre location. Je ne pensais pas que cela serait si compliqué d’acheter un bien</cite>», confie la jeune maman. En cause : les taux élevés et la frilosité des banques à l’octroi de prêts.
        </p>
      </div>
    </section>

    <section>
      <div class="col-medium left-centered-text">
        <p>
          Prenons l’exemple de Sara et son conjoint.  Leur apport comprend d’une part,  46 000 € pour un prêt à taux fixe sur 25 ans avec une quotité de 90%. D’autre part, les frais de notaires et d’enregistrements portent l’apport total à environs 140 000 euros. Une somme difficile à sortir de la poche des jeunes parents. 
        </p>
        <p>
          «<cite> J’avais commencé à économiser dès mon premier salaire et je disposais déjà de 40 000€. C’était largement insuffisant et nous avons dû compter sur les donations de la famille de mon conjoint pour couvrir le reste de la somme. Mes parents sont loin d’être en difficulté mais ils n’auraient pas pu me prêter cet argent. »
        </p>
      </div>
    </section>

    <section>
      <div class="col-medium left-centered-text">
        <p>
          En effet, un tel soutien, nombreuses sont les familles à ne pas pouvoir l’offrir. Sara le confirme : « <cite>Dans la même période, une amie s’est renseignée sur l’achat d’un appartement pour elle et son bébé. En tant que mère célibataire et sans aucun apport extérieur,  la banque lui a proposé 160 000 € de prêt maximum. Un appartement avec deux chambres à ce prix-là, ça n’existe pas.</cite> » Pas un neuf, du moins. 
        </p>
      </div>
    </section>
  </div>
</Scroller>

<Section>
  <h2 class="section-title">Existe-t-il un «facteur jeunesse» ?</h2>

  <p class="mb">
  Il est vrai que les banques se montrent de plus en plus frileuse à accorder des prêts aux jeunes ménages. Fanny Bugeja-Bloch, Maitresse de conférence à l’Université Paris-Nanterre et sociolgue des inégalités analyse cette tendance: «<cite>Les moins de 30 ans sont une population fragile. Ils présentent des signaux qui rendent difficile l’accès à un logement abordable. La précarisation sur le marché, la proportion forte de chômeurs et d’inactifs ou encore les salaires irréguliers sont autant d’éléments qui freinent les banques avant d’accorder un crédit.</cite>»
  </p>

  <div class="conteneur">
      <img src="./img/fanny.png" alt="Fanny">
  </div>


  <p class="mb">
    Tous ces facteurs réunis finissent par plomber le capital dont dipose la jeunesse. Alors qu’en 1985, les moins de 25 ans consacraient au logement un budget deux fois plus élevés qu’un ménage de 75 ans, ce rapport était de 3 en 2015. Des efforts plus importants qui influencent le parcours de vie de ceux qui n’ont pas les ressources suffisantes, souvent héritées de leur famille. Et c'est en fin de vie que les inégalités se font alors ressentir, confirme Fanny Bugeja-Bloch: «<cite>Arrivés à l’âge de la retraite, les locataires ont une plus grande chance de connaitre la précarité car ils n’auront souvent disposé  d’aucun autre moyen de capitalisation pendant leurs années actives.</cite>»
  </p>

</Section>

<Section>
  <h2 class="section-title">Les fonds publics sous pression</h2>

  <p class="mb">
    Sans le soutien des banques et sans famille pour financer leur projet immobilier, les jeunes acheteurs se tournent alors vers les pouvoirs publics. Mais face à l’augmentation du nombre de demandes, leur capacité d'action est de plus en plus limitée. Exemple récent: le Fonds du Logement bruxellois a annoncé ce 2 mai <a href="https://fonds.brussels/fr/a-propos/communiques-de-presse/le-fonds-met-en-place-de-nouvelles-conditions-dacces-pour-son-credit" target="_blank">la restriction des conditions d’accès à ses crédits hypothécaires</a>, désormais réservés aux ménages dont les revenus correspondent aux barèmes du logement social, et ce jusqu’au 31 décembre 2023. 
  </p>


  <p class="mb">
    Mis en place par la région bruxelloise, le Fonds propose notamment des crédits aux conditions avantageuses pour les foyers qui ne disposent pas d’assez de ressources pour contracter un prêt chez une banque. Fort de son succès , le Fonds du logement avait élargi cette possibilité à des foyers de la classe moyenne en 2015. L’institution vient donc de faire marche arrière, pour un temps du moins.
  </p>

  <p>
    les perspectives d'une amélioration sont minces. Rien n'indique que les prix de l'immobilier devraient se maintenir à moyen termes. De leur côté, les taux devraient continuer leur ascension. La Banque centrale européenne vient de relever ses taux de 0,25 points et a annoncé poursuivre sa stratégie de lutte contre l'inflation pour quelques mois encore, entrainant avec elle l'ensemble des banques et éloignant toujours plus de jeunes foyers d’un accès à la propriété.
  </p>

</Section>

<Section>
  <h3>
      Etienne Daman
  </h3>
</Section>

<ONSFooter />






<style>

.dezoomer-button {
    background-color: #007BFF;
    color: white;
    padding: 8px 15px;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-weight: bold;
    font-size: 1rem;
    transition: background-color 0.3s;
}

.dezoomer-button:hover {
    background-color: #0056b3;
}


.conteneur {
    display: flex;
    justify-content: center; /* Centrer horizontalement */
    align-items: center;     /* Centrer verticalement */          /* Exemple de hauteur, à adapter selon vos besoins */
}



.img {
    max-width: 100%;         /* Pour s'assurer que l'image ne déborde pas du conteneur */
    height: auto;
}


.left-centered-text {
    margin-left: 50%; /* Pousse le texte vers la gauche */
}

/* Styles specific to elements within the demo */
:global(svelte-scroller-foreground) {
  pointer-events: none !important;
}

:global(svelte-scroller-foreground section div) {
  pointer-events: all !important;
}


select {
  max-width: 350px;
}

.chart {
  margin-top: 45px;
  width: calc(100% - 5px);
}

.chart-full {
  margin: 0 20px;
}

.chart-sml {
  font-size: 0.85em;
}

:global(.tooltipdecodeurs) { 
    font-family: "Marr Sans", "Helvetica Neue", Helvetica, Arial;
    border-radius: 0;
}
:global(.tooltipdecodeurs .titre_tt) { 
    font-family: "Marr Sans", "Helvetica Neue", Helvetica, Arial;
    font-weight: 600;
    font-size: 110%;
    margin: .5em 0; 
}
.tooltipdecodeurs .gris { color: #a2a9ae; }
.tooltipdecodeurs .bulle { 
    border-radius: 50%;
    width: 1rem;
    height: 1rem;
    border: 0;
    display: inline-block;
    margin: -.3rem .5rem 0 0;
    vertical-align: middle;
    cursor: default; 
}
:global(.tooltipdecodeurs div) { display: block !important; }
:global(.tooltipdecodeurs.hr) { 
    margin-top: 1rem;
    border-top: 1px solid #050505;
    padding-bottom: 1rem; 
}
.tooltipdecodeurs, .arrow:after { 
    background: #FFF;
    border: 1px solid #E2E4E9 !important; 
}

.citation-emphase {
    font-size: 36px; /* ou toute autre taille qui vous semble appropriée */
    font-style: italic; 
    margin: 20px 0;
    padding: 10px;
    border-left: 3px solid #ccc;
    color: #333;
    background-color: #a1cbe3;
}

.attribution {
    font-size: 18px;
    font-style: normal;
    text-align: right;
}

/* The properties below make the media DIVs grey, for visual purposes in demo */

.custom-slider {
    flex-grow: 1;
    margin: 0 10px;
    -webkit-appearance: none;
    height: 5px;
    background: #ddd;
    outline: none;
    border-radius: 5px;
}

.custom-slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 20px;
    height: 20px;
    background: #555;
    cursor: pointer;
    border-radius: 50%;
}

.custom-slider::-moz-range-thumb {
    width: 20px;
    height: 20px;
    background: #555;
    cursor: pointer;
    border-radius: 50%;
}
.custom-slider-container {
    width: 100%; /* Prend toute la largeur disponible de son parent */
    padding: 10px 0; /* Ajoute un peu d'espace au-dessus et en dessous du slider */
    box-sizing: border-box; /* Assurez-vous que les marges et les bordures ne débordent pas */
}

.info-container {
  display: flex;
  gap: 20px; /* Espace entre les cases */
}

.info-box {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.data-box {
  border: 1px solid #ccc; /* Bordure pour la case */
  padding: 5px 15px;      /* Espacement interne pour la case */
  text-align: center;     /* Centrer le texte dans la case */
  margin-top: 5px;        /* Espacement entre le label et la case */
}

cite, blockquote {
    font-style: italic;
}

.limited-width {
    max-width: 600px; /* Par exemple */
}

</style>
