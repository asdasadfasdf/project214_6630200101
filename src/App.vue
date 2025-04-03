<template>
  <div :class="['profile-container', { 'edit-mode': isEditMode }]">
    <!-- Mode -->
    <div class="edit-button  mb-3">
      <button class="btn btn-outline-secondary" @click="toggleEditMode">
        {{ isEditMode ? 'บันทึก' : 'แก้ไขข้อมูลส่วนตัว' }}
      </button>
    </div>

    <!-- Profile Box -->
    <div class="Profile text-center">


      <img v-if="profile.imagePath" :src="profile.imagePath" alt="Profile" class="profile-pic"
        @error="handleImageError" />
      <div v-else class="no-image">ไม่มีรูปภาพ</div>


      <!-- View Mode -->
      <template v-if="!isEditMode">
        <h1>{{ profile.fullname }}</h1>
        <p>รหัสนักศึกษา: {{ profile.studentId }}</p>
        <p>สาขา: {{ profile.major }}</p>
        <p>โรงเรียนเดิม: {{ profile.school }}</p>
      </template>

      <!-- Edit Mode -->
      <template v-else>
        <form @submit.prevent="saveProfile">
          <div class="row g-3 text-start">
            <div class="col-md-6">
              <label class="form-label">ชื่อ - นามสกุล</label>
              <input type="text" class="form-control" v-model="profile.fullname" />
            </div>
            <div class="col-md-6">
              <label class="form-label">รหัสนักศึกษา</label>
              <input type="text" class="form-control" v-model="profile.studentId" />
            </div>
            <div class="col-md-6">
              <label class="form-label">สาขา</label>
              <input type="text" class="form-control" v-model="profile.major" />
            </div>
            <div class="col-md-6">
              <label class="form-label">โรงเรียนเดิม</label>
              <input type="text" class="form-control" v-model="profile.school" />
            </div>
          </div>
          <button type="submit" class="btn btn-primary mt-3">บันทึก</button>
        </form>
      </template>
    </div>

    <!-- Box Group: รายวิชา -->
    <div class="box-group mt-4">
      <div v-for="(terms, year) in subjectData" :key="year">
        <div class="box" v-for="(subjectList, term) in getSortedTerms(terms)" :key="term">
          <div class="box-header">
            <strong>ปีการศึกษา {{ year }} {{ term }}</strong>
            <small class="text-muted">
              หน่วยกิต: {{ calculateCredits(subjectList) }} |
              เกรดเฉลี่ย: {{ calculateGPA(subjectList) }}
            </small>
          </div>
          <table class="table table-sm table-bordered">
            <thead>
              <tr>
                <th>รหัสวิชา</th>
                <th>ชื่อวิชา</th>
                <th>หน่วยกิต</th>
                <th>เกรด</th>
                <th v-if="isEditMode">จัดการ</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(subject, index) in subjectList" :key="index">
                <template v-if="isEditing(year, term, index)">
                  <td><input v-model="editedSubject.code" /></td>
                  <td><input v-model="editedSubject.name" /></td>
                  <td><input v-model="editedSubject.credit" /></td>
                  <td><input v-model="editedSubject.grade" /></td>
                  <td>
                    <button class="btn btn-success btn-sm" @click="saveEditSubject(year, term, index)">บันทึก</button>
                    <button class="btn btn-secondary btn-sm" @click="cancelEdit">ยกเลิก</button>
                    <button class="btn btn-danger btn-sm" @click="removeSubject(year, term, index)">ลบ</button>
                  </td>
                </template>
                <template v-else>
                  <td>{{ subject.code }}</td>
                  <td>{{ subject.name }}</td>
                  <td>{{ subject.credit }}</td>
                  <td>{{ subject.grade }}</td>
                  <td v-if="isEditMode">
                    <button class="btn btn-warning btn-sm"
                      @click="EditSubject(year, term, index, subject)">แก้ไข</button>
                    <button class="btn btn-danger btn-sm" @click="removeSubject(year, term, index)">ลบ</button>
                  </td>
                </template>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- Top Summary and Control Row -->
    <div class="summary-box">
      <h5>สรุปภาพรวม</h5>
      <p>หน่วยกิตสะสม: {{ totalCredits }}</p>
      <p>เกรดเฉลี่ยสะสม (GPA): {{ totalGPA }}</p>
    </div>


    <div class="education-box">
      <div class="edu-images">
        <template v-if="profile.schoolImage">
          <img :src="profile.schoolImage" alt="School" class="edu-img" @error="profile.schoolImage = ''" />
        </template>
        <template v-else>
          <div class="no-image">ไม่มีรูปภาพ</div>
        </template>

        <template v-if="profile.uniImage">
          <img :src="profile.uniImage" alt="University" class="edu-img" @error="profile.uniImage = ''" />
        </template>
        <template v-else>
          <div class="no-image">ไม่มีรูปภาพ</div>
        </template>
      </div>
      <p>รูปภาพโรงเรียนและมหาวิทยาลัย</p>
    </div>

    <!-- Control Panel -->
    <div class="control mb-4" v-if="isEditMode">
      <form class="row g-2 mb-3" @submit.prevent="addSubjectToSelected">
        <div class="col-md-2">
          <input type="text" class="form-control" placeholder="ปีการศึกษา" v-model="selectedYearForAdd" />
        </div>
        <div class="col-md-2">
          <select class="form-select" v-model="selectedTermForAdd">
            <option value="เทอมต้น">เทอมต้น</option>
            <option value="เทอมปลาย">เทอมปลาย</option>
          </select>
        </div>
        <div class="col-md-2">
          <input type="text" class="form-control" placeholder="รหัสวิชา" v-model="newSubject.code" />
        </div>
        <div class="col-md-2">
          <input type="text" class="form-control" placeholder="ชื่อวิชา" v-model="newSubject.name" />
        </div>
        <div class="col-md-1">
          <input type="number" class="form-control" placeholder="หน่วยกิต" v-model="newSubject.credit" />
        </div>
        <div class="col-md-1">
          <input type="text" class="form-control" placeholder="เกรด" v-model="newSubject.grade" />
        </div>
        <div class="col-md-2">
          <button type="submit" class="btn btn-success w-100">เพิ่ม</button>
        </div>
      </form>
      <form class="row g-2 mb-4" @submit.prevent="removeTerm(selectedYearToRemove, selectedTermToRemove)">
        <div class="col-md-3">
          <input type="text" class="form-control" placeholder="ปีที่ต้องการลบ" v-model="selectedYearToRemove" />
        </div>
        <div class="col-md-3">
          <select class="form-select" v-model="selectedTermToRemove">
            <option value="เทอมต้น">เทอมต้น</option>
            <option value="เทอมปลาย">เทอมปลาย</option>
          </select>
        </div>
        <div class="col-md-2">
          <button type="submit" class="btn btn-danger w-100">ลบปีและเทอม</button>
        </div>
        <div class="mt-2" v-if="lastDeletedSubject">
          <button class="btn btn-outline-primary btn-sm" @click="undoDelete">กู้คืนวิชาที่ลบล่าสุด</button>
        </div>
      </form>
    </div>
  </div>


</template>

<script>
import axios from 'axios';
export default {
  data() {
    return {
      profile: {},
      subjectData: {},
      isEditMode: false,
      imagePath: '',
      editingIndex: null,
      editedSubject: {
        code: '',
        name: '',
        credit: '',
        grade: ''
      },
      newSubject: {
        code: '',
        name: '',
        credit: '',
        grade: ''
      },
      selectedYearForAdd: '',
      selectedTermForAdd: 'เทอมต้น',
      selectedYearToRemove: '',
      selectedTermToRemove: '',
      lastDeletedSubject: null,
      lastDeletedMeta: { year: null, term: null, index: null }
    };

  },
  created() {
    // โหลดข้อมูล profile
    axios.get('http://localhost:3000/profile')
      .then(res => {
        this.profile = res.data;
      })
      .catch(err => console.error('โหลด profile ไม่สำเร็จ:', err));

    // โหลดข้อมูล subjectData
    axios.get('http://localhost:3000/subjectData')
      .then(res => {
        this.subjectData = res.data;
      })
      .catch(err => console.error('โหลด subjectData ไม่สำเร็จ:', err));
  },

  computed: {
    totalCredits() {
      let total = 0;
      for (const year of Object.values(this.subjectData)) {
        for (const term of Object.values(year)) {
          total += this.calculateCredits(term);
        }
      }
      return total;
    },
    totalGPA() {
      let totalCredits = 0;
      let weightedGrades = 0;
      for (const year of Object.values(this.subjectData)) {
        for (const term of Object.values(year)) {
          for (const subject of term) {
            const credit = parseFloat(subject.credit);
            const grade = this.gradeToPoint(subject.grade);
            if (!isNaN(credit) && !isNaN(grade)) {
              totalCredits += credit;
              weightedGrades += credit * grade;
            }
          }
        }
      }
      return totalCredits ? (weightedGrades / totalCredits).toFixed(2) : '0.00';
    }
  },
  methods: {
    addSubjectToSelected() {
      const year = this.selectedYearForAdd;
      const term = this.selectedTermForAdd;

      if (!year || !term) {
        alert('กรุณาระบุปีการศึกษาและเทอม');
        return;
      }

      if (term === 'เทอมปลาย' && (!this.subjectData[year] || !this.subjectData[year]['เทอมต้น'])) {
        alert('กรุณาเพิ่มเทอมต้นของปีการศึกษานี้ก่อน');
        return;
      }

      if (!this.subjectData[year]) this.subjectData[year] = {};
      if (!this.subjectData[year][term]) this.subjectData[year][term] = [];

      const subject = { ...this.newSubject };

      if (!subject.code || !subject.name || !subject.credit || !subject.grade) {
        alert('กรุณากรอกข้อมูลให้ครบ');
        return;
      }
      if (this.validateSubject(subject)) {
        subject.grade = this.convertGrade(subject.grade);
        this.subjectData[year][term].push(subject);
        this.newSubject = { code: '', name: '', credit: '', grade: '' };
      }
      this.updateSubjectDataToDB();
    },

    addSubject(year, term) {
      const { code, name, credit, grade } = this.newSubject;
      if (code && name && credit && grade) {
        this.subjectData[year][term].push({ code, name, credit, grade });
        this.newSubject = { code: '', name: '', credit: '', grade: '' };
      }
      this.updateSubjectDataToDB();
    },

    removeSubject(year, term, index) {
      this.lastDeletedSubject = this.subjectData[year][term][index];
      this.lastDeletedMeta = { year, term, index };

      this.subjectData[year][term].splice(index, 1);
      this.updateSubjectDataToDB();

    },

    undoDelete() {
      const { year, term, index } = this.lastDeletedMeta;
      if (!this.subjectData[year]) this.$set(this.subjectData, year, {});
      if (!this.subjectData[year][term]) this.$set(this.subjectData[year], term, []);

      this.subjectData[year][term].splice(index, 0, this.lastDeletedSubject);
      this.updateSubjectDataToDB();

      this.lastDeletedSubject = null;
      this.lastDeletedMeta = { year: null, term: null, index: null };

    },

    isEditing(y, t, i) {
      return (
        this.editingIndex &&
        this.editingIndex.year === y &&
        this.editingIndex.term === t &&
        this.editingIndex.index === i
      );
    },

    EditSubject(year, term, index, subject) {
      this.editingIndex = { year, term, index };
      this.editedSubject = { ...subject };
    },

    cancelEdit() {
      this.editingIndex = null;
      this.editedSubject = {
        code: '',
        name: '',
        credit: '',
        grade: ''
      };
    },

    saveEditSubject(year, term, index) {
      if (!this.isEditMode) {
        this.deletedSubjects = [];
      }
      if (!this.validateSubject(this.editedSubject)) return;

      if (!this.subjectData[year]) {
        this.$set(this.subjectData, year, {});
      }
      if (!this.subjectData[year][term]) {
        this.$set(this.subjectData[year], term, []);
      }

      const subject = { ...this.editedSubject };
      if (this.shouldConvertGrade(subject.grade)) {
        subject.grade = this.convertGrade(parseFloat(subject.grade));
      }

      this.subjectData[year][term].splice(index, 1, subject);
      this.updateSubjectDataToDB();

      this.cancelEdit();
    },

    shouldConvertGrade(grade) {
      const allowed = ['A', 'B+', 'B', 'C+', 'C', 'D+', 'D', 'F'];
      return !allowed.includes(grade) && !isNaN(parseFloat(grade));
    },

    removeTerm(year, term) {
      if (!year || !term) {
        alert('กรุณาระบุปีและเทอมที่ต้องการลบ');
        return;
      }

      if (!this.subjectData[year] || !this.subjectData[year][term]) {
        alert(`ไม่พบข้อมูล ปี ${year} เทอม ${term}`);
        return;
      }

      if (confirm(`คุณแน่ใจหรือไม่ว่าต้องการลบ ปี ${year} เทอม ${term} ?`)) {
        delete this.subjectData[year][term];

        if (Object.keys(this.subjectData[year]).length === 0) {
          delete this.subjectData[year];
        }

        this.updateSubjectDataToDB();

        this.selectedYearToRemove = '';
        this.selectedTermToRemove = '';
      }
    },
    
    getSortedTerms(terms) {
      if (!terms) return {};
      const order = { 'เทอมต้น': 1, 'เทอมปลาย': 2 };
      return Object.keys(terms)
        .sort((a, b) => order[a] - order[b])
        .reduce((sorted, key) => {
          sorted[key] = terms[key];
          return sorted;
        }, {});
    },

    updateSubjectDataToDB() {
      axios.put('http://localhost:3000/subjectData', this.subjectData)
        .then(() => console.log('อัปเดต subjectData สำเร็จ'))
        .catch(err => console.error('error update subjectData:', err));
    },
    
    toggleEditMode() {
      this.isEditMode = !this.isEditMode;
      if (!this.isEditMode) {
        this.editingIndex = null;
        this.deletedSubjects = [];
        this.editedSubject = {
          code: '',
          name: '',
          credit: '',
          grade: '',
          totalCredits: '',
          totalGPA: '',
        };
      }
    },

    handleImageError() {
      this.profile.imagePath = '';
    },

    validateSubject(subject) {
      const allowedLetterGrades = ['A', 'B+', 'B', 'C+', 'C', 'D+', 'D', 'F'];

      let credit = parseFloat(subject.credit);
      let grade = subject.grade;
      let numericGrade = parseFloat(grade);

      if (isNaN(credit) || credit <= 0) {
        alert('หน่วยกิตต้องมากกว่า 0');
        return false;
      }

      if (isNaN(numericGrade) && !allowedLetterGrades.includes(grade)) {
        alert('กรุณากรอกเกรดให้ถูกต้อง (0-4 หรือ A, B+, B, C+, C, D+, D, F)');
        return false;
      }

      return true;
    },

    convertGrade(score) {
      if (score == 4) return 'A';
      if (score >= 3.5) return 'B+';
      if (score >= 3) return 'B';
      if (score >= 2.5) return 'C+';
      if (score >= 2) return 'C';
      if (score >= 1.5) return 'D+';
      if (score >= 1.0) return 'D';
      return 'F';
    },
    saveProfile() {
      axios.put('http://localhost:3000/profile', this.profile)
        .then(() => {
          alert('บันทึกข้อมูลส่วนตัวเรียบร้อยแล้ว');
          this.toggleEditMode();
        })
        .catch(err => {
          console.error('เกิดข้อผิดพลาดในการบันทึกข้อมูล:', err);
          alert('เกิดข้อผิดพลาด');
        });
    },
    calculateCredits(subjectList) {
      return subjectList.reduce((sum, sub) => sum + parseFloat(sub.credit || 0), 0);
    },
    calculateGPA(subjectList) {
      let totalCredits = 0;
      let weightedGrades = 0;
      for (const subject of subjectList) {
        const credit = parseFloat(subject.credit);
        const grade = this.gradeToPoint(subject.grade);
        if (!isNaN(credit) && !isNaN(grade)) {
          totalCredits += credit;
          weightedGrades += credit * grade;
        }
      }
      return totalCredits ? (weightedGrades / totalCredits).toFixed(2) : '0.00';
    },
    gradeToPoint(grade) {
      const map = {
        'A': 4, 'B+': 3.5, 'B': 3,
        'C+': 2.5, 'C': 2, 'D+': 1.5,
        'D': 1, 'F': 0
      };
      if (!isNaN(parseFloat(grade))) return parseFloat(grade);
      return map[grade] ?? NaN;
    }
  }
}
</script>
<style scoped>
html,
body,
#app {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow-x: hidden;
  box-sizing: border-box;
}

:global(body) {
  background-image: url('/flowing_particles_background.jpg');
  background-size: cover;
  background-repeat: no-repeat;
  background-attachment: fixed;
  background-position: center;
}

.profile-container {
  display: grid;
  grid-template-areas:
    "edit-button edit-button edit-button"
    "profile summary control"
    "education education education"
    "box-group box-group box-group";
  grid-template-columns: 1fr 2fr 1fr;
  gap: 24px;
  padding: 24px;

  width: 100vw;
  max-width: 1280px;
  margin: 0 auto;
  min-height: 100vh;
  box-sizing: border-box;

  border-radius: 16px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(6px);

  transition: background 0.9s ease-in-out;
  background: linear-gradient(to bottom right,
      rgba(255, 225, 253, 0.9),
      rgba(0, 17, 173, 0.7));
}

.profile-container.edit-mode {
  background: linear-gradient(to bottom right,
      rgba(240, 255, 240, 0.95),
      rgba(60, 255, 60, 0.7));
}

.edit-button {
  grid-area: edit-button;
  justify-self: start;
}

.Profile {
  grid-area: profile;
  background: #ffffff;
  border: 2px solid #000;
  border-radius: 20px;
  padding: 24px;
  box-sizing: border-box;
  max-width: 100%;
}

.summary-box {
  grid-area: summary;
  background: #ffffff;
  border: 2px solid #000;
  border-radius: 12px;
  padding: 16px;
  text-align: center;
  align-self: start;
  max-width: 100%;
}

.education-box {
  grid-area: education;
  background: linear-gradient(to right, #f8f9ff, #e9e3ff);
  border: 2px solid #000;
  border-radius: 12px;
  padding: 16px;
  text-align: center;
  margin-top: 16px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.05);
}

.edu-images {
  display: flex;
  justify-content: center;
  gap: 16px;
  margin-bottom: 12px;
}

.edu-img {
  width: 150px;
  height: auto;
  object-fit: contain;
  background-color: white;
  border-radius: 8px;
  border: 1px solid #ccc;
}

.control {
  grid-area: control;
  background: #ffffff;
  border: 2px solid #000;
  border-radius: 28px;
  padding: 24px;
  text-align: center;
  max-width: 100%;
  align-self: start;
}

.box-group {
  grid-area: box-group;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 24px;
  padding: 20px;

}

.box {
  background-color: rgb(255, 255, 255);
  border: 2px solid #000;
  border-radius: 12px;
  padding: 24px;
  margin: 12px;
  box-sizing: border-box;

  font-size: 16px;
  text-align: center;

  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

td input {
  width: 100%;
  padding: 4px;
  box-sizing: border-box;
}

td button {
  margin-left: 6px;
  white-space: nowrap;
}

tr.editing td {
  vertical-align: middle;
}

tr.editing td:last-child {
  display: flex;
  gap: 8px;
  justify-content: center;
}

.text-start {
  text-align: left !important;
}

.text-end {
  text-align: right !important;
}

.code-col {
  width: 140px;
  white-space: nowrap;
}

.profile-pic {
  width: 150px;
  height: 150px;
  border-radius: 50%;
  object-fit: cover;
  display: block;
  margin: 0 auto;
  border: 2px solid #ccc;
}

.no-image {
  width: 150px;
  height: auto;
  border-radius: 12px;
  border: 1px solid #ccc;
  object-fit: contain;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #fff;
}
</style>