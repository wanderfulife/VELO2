<template>
  <div class="admin-dashboard-root">
    <button class="btn-signin" @click="showSignInModal = true">
      Connexion Admin
    </button>

    <!-- Sign In Modal -->
    <div v-if="showSignInModal" class="modal">
      <div class="modal-content signin-modal">
        <button class="close" @click="showSignInModal = false">&times;</button>
        <h2>Connexion Admin</h2>
        <input
          v-model="password"
          type="password"
          placeholder="Entrez le mot de passe"
          class="form-control"
        />
        <button @click="signIn" class="btn-signin">Se connecter</button>
      </div>
    </div>

    <!-- Admin Dashboard Modal -->
    <div v-if="showAdminDashboard" class="modal">
      <div class="modal-content admin-dashboard">
        <button class="close" @click="showAdminDashboard = false">
          &times;
        </button>
        <h2>Tableau de Bord Admin</h2>
        <div class="dashboard-content">
          <div class="dashboard-card total">
            <h3>Total des Enquêtes</h3>
            <p class="big-number">{{ totalSurveys }}</p>
          </div>
          <div class="dashboard-card">
            <h3>Enquêtes par Enquêteur</h3>
            <ul>
              <li v-for="(count, name) in surveysByEnqueteur" :key="name">
                <span>{{ name }}</span>
                <span class="count">{{ count }}</span>
              </li>
            </ul>
          </div>

        </div>
        <button @click="downloadData" class="btn-download">
          Télécharger les Données
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { collection, getDocs } from "firebase/firestore";
import { db } from "../firebaseConfig";
import * as XLSX from "xlsx";
import { idfMobilitesCycleSurveyQuestions } from "./idfMobilitesCycleSurveyQuestions.js";

const props = defineProps({
  activeFirebaseCollectionName: {
    type: String,
    required: true,
  },
});

const showSignInModal = ref(false);
const showAdminDashboard = ref(false);
const password = ref("");
const surveysByEnqueteur = ref({});
const totalSurveys = ref(0);

const surveyCollectionRef = collection(db, props.activeFirebaseCollectionName);

const signIn = () => {
  if (password.value === "admin123") {
    showSignInModal.value = false;
    fetchAdminData();
    showAdminDashboard.value = true;
  } else {
    alert("Mot de passe incorrect");
  }
};

const fetchAdminData = async () => {
  try {
    const querySnapshot = await getDocs(surveyCollectionRef);
    const surveys = querySnapshot.docs.map((doc) => doc.data());

    totalSurveys.value = surveys.length;

    surveysByEnqueteur.value = surveys.reduce((acc, survey) => {
      acc[survey.ENQUETEUR] = (acc[survey.ENQUETEUR] || 0) + 1;
      return acc;
    }, {});
  } catch (error) {
    console.error("Erreur lors de la récupération des données :", error);
  }
};

const downloadData = async () => {
  try {
    const querySnapshot = await getDocs(surveyCollectionRef);

    // Store both document data and ID
    const rawData = querySnapshot.docs.map((doc) => ({
      ...doc.data(),
      firestore_id: doc.id, // Add the Firestore document ID
    }));

    // Log raw data for debugging
    console.log("Raw survey data from Firestore:", rawData);

    // Define core headers that should appear first and in this order
    const coreHeaders = [
      "ID_questionnaire",
      "ENQUETEUR",
      "DATE",
      "JOUR",
      "HEURE_DEBUT",
      "HEURE_FIN",
      "POSTE_TRAVAIL"
    ];

    // Define keys to be completely excluded from the export
    const excludedKeys = ["firestore_id", "firebase_timestamp", "S1"];

    // Get the survey question order from the IDF survey questions
    const surveyQuestionOrder = idfMobilitesCycleSurveyQuestions.map(q => q.id);
    
    // Collect all keys that exist in the data
    let allKeys = new Set(coreHeaders);
    rawData.forEach(docData => {
      Object.keys(docData).forEach(key => {
        if (!excludedKeys.includes(key)) {
          allKeys.add(key);
        }
      });
    });

    // Create ordered headers: core headers first, then survey questions in order, then any remaining keys
    const surveyHeaders = surveyQuestionOrder.filter(questionId => 
      allKeys.has(questionId) && 
      !coreHeaders.includes(questionId)
    );
    
    const remainingHeaders = Array.from(allKeys)
      .filter(key => 
        !coreHeaders.includes(key) && 
        !surveyQuestionOrder.includes(key) && 
        !excludedKeys.includes(key)
      )
      .sort(); // Sort remaining headers alphabetically
    
    // Rename POSTE_TRAVAIL to POSTE in the header order
    const headerOrder = [...coreHeaders, ...surveyHeaders, ...remainingHeaders].map(header => 
      header === "POSTE_TRAVAIL" ? "POSTE" : header
    );

    const data = rawData.map((docData) => {
      const processedData = {};
      // Ensure all headers are present in each row, with empty string for missing ones
      for (const header of headerOrder) {
        // Double check not to include excluded keys, though headerOrder should already be filtered
        if (!excludedKeys.includes(header)) {
            // Map POSTE back to POSTE_TRAVAIL for data lookup
            const dataKey = header === "POSTE" ? "POSTE_TRAVAIL" : header;
            let value = docData[dataKey] !== undefined ? docData[dataKey] : "";
            
            // Handle arrays (from multiple choice questions) by converting to comma-separated string
            if (Array.isArray(value)) {
                value = value.join(", ");
            }
            
            processedData[header] = value;
        }
      }
      return processedData;
    });

    // Log processed data for debugging
    console.log("Processed data for Excel:", data);

    // Create workbook and add main worksheet
    const workbook = XLSX.utils.book_new();
    const worksheet = XLSX.utils.json_to_sheet(data, { header: headerOrder });

    // Set column widths
    const colWidths = headerOrder.map(() => ({ wch: 20 }));
    worksheet["!cols"] = colWidths;
    XLSX.utils.book_append_sheet(workbook, worksheet, "Survey Data");

    // Use a timestamp in the filename to avoid overwriting
    const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
    XLSX.writeFile(workbook, `${props.activeFirebaseCollectionName}_Survey_Data_${timestamp}.xlsx`);

    console.log("File downloaded successfully");
  } catch (error) {
    console.error("Error downloading data:", error);
  }
};

onMounted(() => {
  // Initialization logic if needed
});
</script>

<style scoped>
.admin-dashboard-root {
  background-color: #2a3b63;
  margin: 0;
  padding: 0;
  border: none;
  outline: none;
}

.btn-signin {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: linear-gradient(135deg, rgba(45, 55, 72, 0.9) 0%, rgba(26, 32, 44, 0.9) 100%);
  color: #a0aec0;
  border: 1px solid rgba(255, 255, 255, 0.1);
  cursor: pointer;
  font-size: 11px;
  font-weight: 400;
  padding: 8px 12px;
  border-radius: 20px;
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  text-transform: lowercase;
  letter-spacing: 0.5px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  opacity: 0.6;
  z-index: 100;
  font-family: 'SF Pro Text', -apple-system, BlinkMacSystemFont, sans-serif;
}

.btn-signin:hover {
  background: linear-gradient(135deg, rgba(59, 130, 246, 0.2) 0%, rgba(37, 99, 235, 0.2) 100%);
  color: #e2e8f0;
  opacity: 1;
  transform: translateY(-2px) scale(1.05);
  box-shadow: 0 8px 25px rgba(59, 130, 246, 0.15);
  border-color: rgba(59, 130, 246, 0.3);
}

.btn-signin:active {
  transform: translateY(0) scale(1.02);
  transition: all 0.1s ease;
}

/* Keep the rest of the styles unchanged */
.btn-download {
  width: 100%;
  padding: 12px;
  background-color: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-top: 10px;
  box-shadow: none;
}

.btn-download:hover {
  background-color: #2563eb;
}

.modal {
  position: fixed;
  z-index: 1000;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: #2d3748;
  color: white;
  padding: 25px;
  border-radius: 15px;
  width: auto;
  min-width: 300px;
  max-width: 600px;
  position: relative;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.4);
  margin: 0 auto;
}

/* Override modal-content when it's also admin-dashboard */
.modal-content.admin-dashboard {
  padding: 10px !important; /* Force smaller padding */
  min-width: 400px !important;
  max-width: 450px !important;
  height: auto !important; /* Force auto height */
  min-height: auto !important; /* Force auto min height */
}

.modal-content.admin-dashboard h2 {
  margin: 0 0 8px 0 !important; /* Force smaller margin */
  font-size: 16px !important; /* Force smaller title */
}

.modal-content.admin-dashboard .dashboard-content {
  gap: 4px !important; /* Force tiny gap */
  margin-bottom: 4px !important; /* Force tiny margin */
}

.modal-content.admin-dashboard .dashboard-card {
  padding: 6px !important; /* Force tiny padding */
  margin: 0 !important; /* Force no margin */
}

.modal-content.admin-dashboard .dashboard-card h3 {
  margin: 0 0 2px 0 !important; /* Force tiny margin */
  font-size: 12px !important; /* Force smaller font */
}

.modal-content.admin-dashboard .big-number {
  font-size: 20px !important; /* Force smaller number */
  margin: 0 !important; /* Force no margin */
}

.modal-content.admin-dashboard .btn-download {
  margin-top: 4px !important; /* Force tiny margin */
  padding: 8px !important; /* Force tiny padding */
  font-size: 12px !important; /* Force tiny font */
}

/* Force signin modal to be compact */
.modal-content.signin-modal {
  padding: 15px !important;
  width: auto !important;
  min-width: 280px !important;
  max-width: 280px !important;
  height: auto !important;
  min-height: auto !important;
}

.signin-modal {
  max-width: 280px !important; /* Force smaller width */
  padding: 0 !important; /* Remove all padding since modal-content has it */
}

.signin-modal h2 {
  margin: 0 0 10px 0 !important; /* Force tiny margin */
  font-size: 16px !important; /* Force smaller title */
  text-align: center;
  font-weight: normal;
}

.signin-modal .form-control {
  width: 100%;
  margin-bottom: 8px !important; /* Force tiny margin */
  background-color: rgba(255, 255, 255, 0.1);
  border: none;
  color: white;
  padding: 8px !important; /* Force tiny padding */
  font-size: 14px;
  border-radius: 6px !important; /* Smaller radius */
  box-sizing: border-box;
}

.signin-modal .form-control::placeholder {
  color: rgba(255, 255, 255, 0.6);
}

.signin-modal .btn-signin {
  width: 100%;
  margin: 0 !important;
  padding: 8px !important; /* Force tiny padding */
  background: linear-gradient(135deg, #68d391 0%, #4fd1c7 100%) !important; /* Beautiful gradient */
  color: white !important;
  border: none !important;
  border-radius: 6px !important; /* Smaller radius */
  font-size: 12px !important; /* Force smaller font */
  font-weight: 600 !important; /* Semi-bold */
  text-transform: uppercase;
  cursor: pointer;
  transition: all 0.3s ease !important;
  box-sizing: border-box;
  box-shadow: 0 3px 8px rgba(104, 211, 145, 0.3) !important; /* Smaller shadow */
  letter-spacing: 0.5px !important; /* Slight letter spacing */
}

.signin-modal .btn-signin:hover {
  background: linear-gradient(135deg, #4fd1c7 0%, #68d391 100%) !important; /* Reverse gradient on hover */
  transform: translateY(-2px) !important; /* Lift effect */
  box-shadow: 0 6px 20px rgba(104, 211, 145, 0.4) !important; /* Enhanced glow */
}

.close {
  position: absolute;
  right: 15px;
  top: 15px;
  width: 24px;
  height: 24px;
  opacity: 0.7;
  background: none;
  border: none;
  cursor: pointer;
  color: white;
  font-size: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: opacity 0.3s;
}

.close:hover {
  opacity: 1;
}

.admin-dashboard {
  width: auto;
  min-width: 500px;
  max-width: 600px;
  padding: 15px 20px;
}

.admin-dashboard h2 {
  margin: 0 0 15px 0;
  font-size: 20px;
  text-align: center;
  font-weight: normal;
  color: white;
}

.dashboard-content {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 8px;
}

.dashboard-card {
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  padding: 10px;
}

.dashboard-card h3 {
  margin: 0 0 4px 0;
  font-size: 14px;
  color: #3b82f6;
  font-weight: normal;
}

.dashboard-card.total {
  text-align: center;
  padding: 8px;
}

.big-number {
  font-size: 28px;
  font-weight: bold;
  color: #68d391;
  margin: 2px 0;
}

.dashboard-card ul {
  list-style-type: none;
  padding: 0;
  margin: 0;
  max-height: 200px;
  overflow-y: auto;
}

.dashboard-card li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 0;
  color: white;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.dashboard-card li:last-child {
  border-bottom: none;
}

.count {
  font-weight: normal;
  color: #68d391;
}

@media (max-width: 600px) {
  .modal-content {
    margin: 10px;
    width: calc(100% - 20px);
    max-height: 90vh;
  }

  .admin-dashboard {
    padding: 15px;
  }

  .admin-dashboard h2 {
    font-size: 20px;
    margin-bottom: 15px;
  }

  .dashboard-card {
    padding: 12px;
  }

  .big-number {
    font-size: 36px;
  }

  .btn-download {
    padding: 12px;
    font-size: 14px;
  }
}
</style>