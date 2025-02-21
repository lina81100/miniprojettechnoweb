<template>
  <div class="card">
    <h2 class="title">Enregistrer une participation</h2>

    <div v-if="errorMessage" class="error-box">
      {{ errorMessage }}
    </div>

    <div class="form-group">
      <label for="personne">Personne</label>
      <select id="personne" v-model="formulaire.personne" class="input">
        <option v-for="p in personnes" :key="p.matricule" :value="p">
          {{ p.nom }} ({{ p.matricule }})
        </option>
      </select>
    </div>

    <div class="form-group">
      <label for="projet">Projet</label>
      <select id="projet" v-model="formulaire.projet" class="input">
        <option v-for="p in projets" :key="p.id" :value="p">
          {{ p.nom }}
        </option>
      </select>
    </div>

    <div class="form-group">
      <label for="role">Rôle</label>
      <input id="role" v-model="formulaire.role" type="text" class="input" placeholder="Développeur">
    </div>

    <!-- Curseur visuel pourcentage -->
    <div class="form-group">
      <label>Pourcentage</label>
      <input
        type="range"
        v-model="pourcentageAffiche"
        min="0"
        max="100"
        step="5"
        @input="mettreAJourPourcentage"
        class="slider"
      >
      <p class="pourcentage">{{ pourcentageAffiche }}%</p>
    </div>

    <button class="btn" @click="enregistrerParticipation">Enregistrer</button>
  </div>
</template>

<script setup>
import { onMounted, reactive, ref } from "vue";

// État des erreurs
const errorMessage = ref("");

// Modèle de formulaire vide
const participationVide = {
  personne: "",
  projet: "",
  role: "",
  pourcentage: 0.0,
};

const pourcentageAffiche = ref(0);

const mettreAJourPourcentage = () => {
  formulaire.pourcentage = pourcentageAffiche.value / 100;
};

// État réactif des données
const formulaire = reactive({ ...participationVide });
const personnes = ref([]);
const projets = ref([]);

async function enregistrerParticipation() {
  errorMessage.value = ""; // Réinitialiser les erreurs avant chaque envoi

  if (!formulaire.personne || !formulaire.projet) {
    errorMessage.value = "Veuillez sélectionner une personne et un projet.";
    return;
  }

  const personneHref = formulaire.personne._links.self.href;
  const projetHref = formulaire.projet._links.self.href;

  // Vérifier si la participation existe déjà
  if (await participationExiste(personneHref, projetHref)) {
    errorMessage.value = "Cette personne participe déjà à ce projet.";
    return;
  }

  try {
    const response = await fetch("/api/participations", {
      method: "POST",
      body: JSON.stringify({
        personne: personneHref,
        projet: projetHref,
        role: formulaire.role,
        pourcentage: formulaire.pourcentage,
      }),
      headers: {
        "Content-Type": "application/json",
      },
    });

    if (!response.ok) {
      const errorData = await response.json();
      handleApiError(response.status, errorData);
      return;
    }

    alert("Participation enregistrée avec succès !");
    Object.assign(formulaire, participationVide); // Réinitialisation du formulaire
    pourcentageAffiche.value = 0;
  } catch (error) {
    errorMessage.value = "Une erreur réseau s'est produite. Vérifiez votre connexion.";
    console.error("Détails de l'erreur :", error);
  }
}

// Gérer les erreurs de l'API avec des messages précis
function handleApiError(status, errorData) {
  switch (status) {
    case 400:
      errorMessage.value = "Erreur de requête : Données invalides.";
      break;
    case 409:
      errorMessage.value = "Conflit : Cette participation existe déjà !";
      break;
    case 500:
      errorMessage.value = "Erreur serveur : Réessayez plus tard.";
      break;
    default:
      errorMessage.value = `Erreur inconnue (${status}) : ${errorData?.message || "Réessayez plus tard."}`;
  }
}

// Charger les personnes et projets
async function refresh() {
  try {
    const resPersonnes = await fetch("/api/personnes");
    const resProjets = await fetch("/api/projets");

    if (!resPersonnes.ok || !resProjets.ok) throw new Error("Erreur de chargement des données");

    const personnesData = await resPersonnes.json();
    const projetsData = await resProjets.json();

    personnes.value = personnesData._embedded.personnes;
    projets.value = projetsData._embedded.projets;
  } catch (error) {
    errorMessage.value = "Impossible de charger les données.";
    console.error(error);
  }
}

// Vérifier si la participation existe déjà
async function participationExiste(personneHref, projetHref) {
  try {
    const response = await fetch("/api/participations");
    if (!response.ok) throw new Error("Erreur lors de la vérification");

    const data = await response.json();
    return data._embedded.participations.some(p =>
      p._links.personne.href === personneHref &&
      p._links.projet.href === projetHref
    );
  } catch (error) {
    errorMessage.value = "Impossible de vérifier l'existence de la participation.";
    console.error("Erreur lors de la vérification :", error);
    return false;
  }
}

onMounted(refresh);
</script>

<style scoped>
.card {
  max-width: 400px;
  margin: 2rem auto;
  padding: 20px;
  background: #f9f9f9;
  border-radius: 12px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  text-align: left;
}

.error-box {
  background: #ffdddd;
  color: #d8000c;
  padding: 10px;
  border-left: 5px solid #d8000c;
  margin-bottom: 15px;
  font-weight: bold;
}

.title {
  text-align: center;
  font-size: 22px;
  font-weight: bold;
  margin-bottom: 20px;
}

.form-group {
  margin-bottom: 15px;
}

label {
  font-weight: bold;
  display: block;
  margin-bottom: 5px;
}

.input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 16px;
}

.slider {
  width: 100%;
  margin-top: 5px;
}

.pourcentage {
  text-align: center;
  font-size: 18px;
  font-weight: bold;
  margin-top: 5px;
}

.btn {
  width: 100%;
  background: #007BFF;
  color: white;
  padding: 12px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
}

.btn:hover {
  background: #0056b3;
}
</style>
