const fs = require('fs');
const path = require('path');

// ==========================================
// 1. Data Source
// ==========================================

const sampleStudents = [
  {
    id: 1,
    name: 'Alice Johnson',
    year: 3,
    course: 'Computer Science',
    grades: [92, 88, 95, 91],
    enrolled: true
  },
  {
    id: 2,
    name: 'Bob Smith',
    year: 1,
    course: 'Information Technology',
    grades: [75, 82, 79, 80],
    enrolled: false
  },
  {
    id: 3,
    name: 'Charlie Brown',
    year: 4,
    course: 'Computer Science',
    grades: [98, 94, 100, 96],
    enrolled: true
  },
  {
    id: 4,
    name: 'Diana Prince',
    year: 2,
    course: 'Data Science',
    grades: [88, 90, 85, 92],
    enrolled: true
  },
  {
    id: 5,
    name: 'Ethan Hunt',
    year: 2,
    course: 'Cybersecurity',
    grades: [65, 70, 68, 72],
    enrolled: true
  },
  {
    id: 6,
    name: 'Fiona Gallagher',
    year: 1,
    course: 'Information Technology',
    grades: [84, 86, 88, 85],
    enrolled: true
  },
  {
    id: 7,
    name: 'George Clark',
    year: 3,
    course: 'Cybersecurity',
    grades: [],
    enrolled: false
  },
  {
    id: 8,
    name: 'Hannah Abbott',
    year: 4,
    course: 'Data Science',
    grades: [95, 97, 93, 99],
    enrolled: true
  }
];

// ==========================================
// 2. Core Analytics Functions
// ==========================================

/**
 * Calculates the average grade for a single student.
 */
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

/**
 * Gets the top n performing students sorted by average grade.
 */
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

/**
 * Groups students by their enrolled course.
 */
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

/**
 * Counts enrolled versus non-enrolled students.
 */
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

/**
 * Performs a case-insensitive search for a student by name.
 */
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

/**
 * Calculates average grade per course department.
 */
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

/**
 * Constructs an overall summary payload.
 */
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
// 3. Execution & File Output
// ==========================================

function main() {
  const reportPath = path.join(__dirname, 'report.json');

  console.log('='.repeat(50));
  console.log('         STUDENT DATA ANALYSIS REPORT          ');
  console.log('='.repeat(50));

  const enrollmentStatus = getEnrolledCount(sampleStudents);
  console.log(`\n[+] Total Students : ${sampleStudents.length}`);
  console.log(`    - Enrolled     : ${enrollmentStatus.enrolled}`);
  console.log(`    - Not Enrolled : ${enrollmentStatus.notEnrolled}`);

  const summary = exportSummary(sampleStudents);
  console.log(`\n[+] Overall Class Average : ${summary.overallAverage}`);

  const topThree = getTopStudents(sampleStudents, 3);
  console.log('\n[+] Top 3 Performing Students:');
  topThree.forEach((s, idx) => {
    console.log(`    ${idx + 1}. ${s.name} (${s.course}) - Avg: ${s.averageGrade.toFixed(2)}`);
  });

  const courseAverages = getCourseAverages(sampleStudents);
  console.log('\n[+] Course Average Grades (Descending):');
  courseAverages.forEach(c => {
    console.log(`    - ${c.course.padEnd(22)} : ${c.averageGrade}`);
  });

  const searchName = 'Alice Johnson';
  const found = findStudent(sampleStudents, searchName);
  console.log(`\n[+] Search Result for "${searchName}":`);
  if (found) {
    console.log(`    Found: ID #${found.id} | Course: ${found.course} | Avg Grade: ${getAverageGrade(found).toFixed(2)}`);
  } else {
    console.log('    No matching student found.');
  }

  try {
    fs.writeFileSync(reportPath, JSON.stringify(summary, null, 2), 'utf8');
    console.log(`\n[+] Summary file created at: "${reportPath}"`);
  } catch (err) {
    console.error('Failed to export summary file:', err.message);
  }

  console.log('\n' + '='.repeat(50));
}

// Run script
main();
