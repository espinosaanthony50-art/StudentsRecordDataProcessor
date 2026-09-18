# 1. Create project folder and step inside
mkdir student-records-processor && cd student-records-processor

# 2. Create app.js
cat << 'EOF' > app.js
const fs = require('fs');
const path = require('path');

// ==========================================
// Core Functions
// ==========================================

function getAverageGrade(student) {
  if (!student || typeof student !== 'object') {
    throw new Error('Invalid input: Expected a student object.');
  }
  if (!Array.isArray(student.grades) || student.grades.length === 0) {
    return 0;
  }
  const total = student.grades.reduce((sum, grade) => sum + grade, 0);
  return total / student.grades.length;
}

function getTopStudents(students, n) {
  if (!Array.isArray(students)) {
    throw new Error('Invalid input: Expected students to be an array.');
  }
  if (typeof n !== 'number' || n < 0 || !Number.isInteger(n)) {
    throw new Error('Invalid input: n must be a non-negative integer.');
  }

  return [...students]
    .map(student => ({
      ...student,
      averageGrade: getAverageGrade(student)
    }))
    .sort((a, b) => b.averageGrade - a.averageGrade)
    .slice(0, n);
}

function groupByCourse(students) {
  if (!Array.isArray(students)) {
    throw new Error('Invalid input: Expected students to be an array.');
  }

  return students.reduce((acc, student) => {
    const course = student.course || 'Unassigned';
    if (!acc[course]) {
      acc[course] = [];
    }
    acc[course].push({ ...student });
    return acc;
  }, {});
}

function getEnrolledCount(students) {
  if (!Array.isArray(students)) {
    throw new Error('Invalid input: Expected students to be an array.');
  }

  return students.reduce(
    (acc, student) => {
      if (student.enrolled) {
        acc.enrolled += 1;
      } else {
        acc.notEnrolled += 1;
      }
      return acc;
    },
    { enrolled: 0, notEnrolled: 0 }
  );
}

function findStudent(students, name) {
  if (!Array.isArray(students)) {
    throw new Error('Invalid input: Expected students to be an array.');
  }
  if (typeof name !== 'string') {
    throw new Error('Invalid input: Name must be a string.');
  }

  const searchTarget = name.trim().toLowerCase();
  const match = students.find(
    s => s.name && s.name.toLowerCase() === searchTarget
  );

  return match ? { ...match } : null;
}

function getCourseAverages(students) {
  if (!Array.isArray(students)) {
    throw new Error('Invalid input: Expected students to be an array.');
  }

  const grouped = groupByCourse(students);
  
  const courseAverages = Object.keys(grouped).map(course => {
    const courseStudents = grouped[course];
    const totalGrades = courseStudents.map(getAverageGrade);
    const avg = totalGrades.length > 0 
      ? totalGrades.reduce((sum, val) => sum + val, 0) / totalGrades.length 
      : 0;
    
    return {
      course,
      averageGrade: Number(avg.toFixed(2))
    };
  });

  return courseAverages.sort((a, b) => b.averageGrade - a.averageGrade);
}

function exportSummary(students) {
  if (!Array.isArray(students)) {
    throw new Error('Invalid input: Expected students to be an array.');
  }

  const totalStudents = students.length;
  
  if (totalStudents === 0) {
    return {
      totalStudents: 0,
      overallAverage: 0,
      topStudent: null,
      courseBreakdown: []
    };
  }

  const totalAverages = students.map(getAverageGrade);
  const overallAvg = totalAverages.reduce((sum, g) => sum + g, 0) / totalStudents;
  const topStudent = getTopStudents(students, 1)[0] || null;

  return {
    totalStudents,
    overallAverage: Number(overallAvg.toFixed(2)),
    topStudent: topStudent ? { name: topStudent.name, averageGrade: Number(topStudent.averageGrade.toFixed(2)) } : null,
    courseBreakdown: getCourseAverages(students)
  };
}

// ==========================================
// Main Runner
// ==========================================

function main() {
  const dataPath = path.join(__dirname, 'students.json');
  const reportPath = path.join(__dirname, 'report.json');

  let students = [];

  try {
    const fileData = fs.readFileSync(dataPath, 'utf8');
    students = JSON.parse(fileData);
  } catch (err) {
    console.error(`Error reading dataset at ${dataPath}:`, err.message);
    process.exit(1);
  }

  console.log('='.repeat(50));
  console.log('         STUDENT DATA ANALYSIS REPORT          ');
  console.log('='.repeat(50));

  const enrollmentStatus = getEnrolledCount(students);
  console.log(`\n[+] Total Students : ${students.length}`);
  console.log(`    - Enrolled     : ${enrollmentStatus.enrolled}`);
  console.log(`    - Not Enrolled : ${enrollmentStatus.notEnrolled}`);

  const summary = exportSummary(students);
  console.log(`\n[+] Overall Class Average : ${summary.overallAverage}`);

  const topThree = getTopStudents(students, 3);
  console.log('\n[+] Top 3 Performing Students:');
  topThree.forEach((s, idx) => {
    console.log(`    ${idx + 1}. ${s.name} (${s.course}) - Avg: ${s.averageGrade.toFixed(2)}`);
  });

  const courseAverages = getCourseAverages(students);
  console.log('\n[+] Course Average Grades (Descending):');
  courseAverages.forEach(c => {
    console.log(`    - ${c.course.padEnd(20)} : ${c.averageGrade}`);
  });

  const searchName = 'Alice Johnson';
  const found = findStudent(students, searchName);
  console.log(`\n[+] Search Result for "${searchName}":`);
  if (found) {
    console.log(`    Found: ID #${found.id} | Course: ${found.course} | Avg Grade: ${getAverageGrade(found).toFixed(2)}`);
  } else {
    console.log('    No matching student found.');
  }

  try {
    fs.writeFileSync(reportPath, JSON.stringify(summary, null, 2), 'utf8');
    console.log(`\n[+] Summary successfully written to "${path.basename(reportPath)}"`);
  } catch (err) {
    console.error('Failed to export summary file:', err.message);
  }

  console.log('\n' + '='.repeat(50));
}

module.exports = {
  getAverageGrade,
  getTopStudents,
  groupByCourse,
  getEnrolledCount,
  findStudent,
  getCourseAverages,
  exportSummary
};

if (require.main === module) {
  main();
}
EOF

# 3. Create app.test.js
cat << 'EOF' > app.test.js
const { test, describe } = require('node:test');
const assert = require('node:assert/strict');

const {
  getAverageGrade,
  getTopStudents,
  groupByCourse,
  getEnrolledCount,
  findStudent,
  getCourseAverages,
  exportSummary
} = require('./app.js');

const sampleStudents = [
  { id: 1, name: 'Alice', course: 'CS', grades: [90, 100], enrolled: true },
  { id: 2, name: 'Bob', course: 'IT', grades: [80, 70], enrolled: false },
  { id: 3, name: 'Charlie', course: 'CS', grades: [], enrolled: true }
];

describe('Student Data Processor Unit Tests', () => {

  describe('getAverageGrade', () => {
    test('calculates correct average for valid grades', () => {
      assert.strictEqual(getAverageGrade(sampleStudents[0]), 95);
    });

    test('returns 0 for empty grades array', () => {
      assert.strictEqual(getAverageGrade(sampleStudents[2]), 0);
    });

    test('throws error for invalid input', () => {
      assert.throws(() => getAverageGrade(null), /Invalid input/);
    });
  });

  describe('getTopStudents', () => {
    test('returns top n students sorted descending', () => {
      const result = getTopStudents(sampleStudents, 2);
      assert.strictEqual(result.length, 2);
      assert.strictEqual(result[0].name, 'Alice');
      assert.strictEqual(result[1].name, 'Bob');
    });

    test('throws error on negative n', () => {
      assert.throws(() => getTopStudents(sampleStudents, -1), /non-negative integer/);
    });
  });

  describe('groupByCourse', () => {
    test('groups students correctly by course name', () => {
      const grouped = groupByCourse(sampleStudents);
      assert.strictEqual(grouped['CS'].length, 2);
      assert.strictEqual(grouped['IT'].length, 1);
    });
  });

  describe('getEnrolledCount', () => {
    test('accurately counts enrolled vs not enrolled', () => {
      const counts = getEnrolledCount(sampleStudents);
      assert.deepStrictEqual(counts, { enrolled: 2, notEnrolled: 1 });
    });
  });

  describe('findStudent', () => {
    test('finds student case-insensitively', () => {
      const student = findStudent(sampleStudents, 'aLiCe');
      assert.strictEqual(student.id, 1);
    });

    test('returns null when student is not found', () => {
      const student = findStudent(sampleStudents, 'Unknown');
      assert.strictEqual(student, null);
    });
  });

  describe('getCourseAverages', () => {
    test('calculates and sorts course averages correctly', () => {
      const courseAvgs = getCourseAverages(sampleStudents);
      assert.strictEqual(courseAvgs[0].course, 'IT');
      assert.strictEqual(courseAvgs[0].averageGrade, 75);
      assert.strictEqual(courseAvgs[1].course, 'CS');
      assert.strictEqual(courseAvgs[1].averageGrade, 47.5);
    });
  });

  describe('exportSummary', () => {
    test('generates expected structure for standard dataset', () => {
      const summary = exportSummary(sampleStudents);
      assert.strictEqual(summary.totalStudents, 3);
      assert.strictEqual(summary.topStudent.name, 'Alice');
    });

    test('handles empty dataset gracefully', () => {
      const summary = exportSummary([]);
      assert.deepStrictEqual(summary, {
        totalStudents: 0,
        overallAverage: 0,
        topStudent: null,
        courseBreakdown: []
      });
    });
  });

});
EOF

# 4. Create students.json
cat << 'EOF' > students.json
[
  {
    "id": 1,
    "name": "Alice Johnson",
    "year": 3,
    "course": "Computer Science",
    "grades": [92, 88, 95, 91],
    "enrolled": true
  },
  {
    "id": 2,
    "name": "Bob Smith",
    "year": 1,
    "course": "Information Technology",
    "grades": [75, 82, 79, 80],
    "enrolled": false
  },
  {
    "id": 3,
    "name": "Charlie Brown",
    "year": 4,
    "course": "Computer Science",
    "grades": [98, 94, 100, 96],
    "enrolled": true
  },
  {
    "id": 4,
    "name": "Diana Prince",
    "year": 2,
    "course": "Data Science",
    "grades": [88, 90, 85, 92],
    "enrolled": true
  },
  {
    "id": 5,
    "name": "Ethan Hunt",
    "year": 2,
    "course": "Cybersecurity",
    "grades": [65, 70, 68, 72],
    "enrolled": true
  },
  {
    "id": 6,
    "name": "Fiona Gallagher",
    "year": 1,
    "course": "Information Technology",
    "grades": [84, 86, 88, 85],
    "enrolled": true
  },
  {
    "id": 7,
    "name": "George Clark",
    "year": 3,
    "course": "Cybersecurity",
    "grades": [],
    "enrolled": false
  },
  {
    "id": 8,
    "name": "Hannah Abbott",
    "year": 4,
    "course": "Data Science",
    "grades": [95, 97, 93, 99],
    "enrolled": true
  }
]
EOF

# 5. Create package.json
cat << 'EOF' > package.json
{
  "name": "student-records-processor",
  "version": "1.0.0",
  "description": "Pure JavaScript Node.js student data analyzer.",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "node --test"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
EOF

# 6. Create README.md
cat << 'EOF' > README.md
# Student Records Data Processor

A pure Node.js data processor that reads student records, performs statistical analysis, and outputs structured console logs along with a exported summary file.

## Features
- Pure JavaScript (no external npm dependencies).
- Functional array manipulation using `map`, `filter`, `reduce`, and `sort`.
- Standard File System (fs) file parsing and export.
- Native unit test suite via Node.js test runner (`node:test`).

## Commands

- **Run Script**:
  ```bash
  npm start
