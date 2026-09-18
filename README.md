const fs = require('fs');
const path = require('path');

// ==========================================
// 1. Data Processor Functions
// ==========================================

/**
 * Calculates the average grade for a single student.
 */
function getAverageGrade(student) {
  if (!student || !Array.isArray(student.grades) || student.grades.length === 0) {
    return 0;
  }
  const sum = student.grades.reduce((acc, curr) => acc + curr, 0);
  return sum / student.grades.length;
}

/**
 * Returns the top n students sorted by average grade descending.
 */
function getTopStudents(students, n) {
  if (!Array.isArray(students)) return [];
  if (typeof n !== 'number' || n < 0) {
    throw new Error('Parameter "n" must be a non-negative number.');
  }

  return [...students]
    .sort((a, b) => getAverageGrade(b) - getAverageGrade(a))
    .slice(0, n);
}

/**
 * Groups all students by their course field.
 */
function groupByCourse(students) {
  if (!Array.isArray(students)) return {};

  return students.reduce((acc, student) => {
    const course = student.course || 'Unassigned';
    if (!acc[course]) {
      acc[course] = [];
    }
    acc[course].push(student);
    return acc;
  }, {});
}

/**
 * Returns count of enrolled vs non-enrolled students.
 */
function getEnrolledCount(students) {
  if (!Array.isArray(students)) return { enrolled: 0, notEnrolled: 0 };

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
 * Case-insensitive student search by name.
 */
function findStudent(students, name) {
  if (!Array.isArray(students) || typeof name !== 'string') return null;

  const targetName = name.trim().toLowerCase();
  return students.find((student) => student.name?.toLowerCase() === targetName) || null;
}

/**
 * Calculates average grade per course, sorted highest to lowest.
 */
function getCourseAverages(students) {
  if (!Array.isArray(students) || students.length === 0) return [];

  const grouped = groupByCourse(students);

  return Object.entries(grouped)
    .map(([course, courseStudents]) => {
      const allGrades = courseStudents.flatMap((s) => s.grades || []);
      const avg =
        allGrades.length > 0
          ? allGrades.reduce((sum, g) => sum + g, 0) / allGrades.length
          : 0;

      return {
        course,
        averageGrade: Number(avg.toFixed(2)),
      };
    })
    .sort((a, b) => b.averageGrade - a.averageGrade);
}

/**
 * Exports summary statistics.
 */
function exportSummary(students) {
  if (!Array.isArray(students) || students.length === 0) {
    return {
      totalStudents: 0,
      overallAverageGrade: 0,
      topStudent: null,
      courseBreakdown: [],
    };
  }

  const allGrades = students.flatMap((s) => s.grades || []);
  const overallAverageGrade =
    allGrades.length > 0
      ? allGrades.reduce((sum, g) => sum + g, 0) / allGrades.length
      : 0;

  const topStudent = getTopStudents(students, 1)[0] || null;

  return {
    totalStudents: students.length,
    overallAverageGrade: Number(overallAverageGrade.toFixed(2)),
    topStudent: topStudent
      ? {
          id: topStudent.id,
          name: topStudent.name,
          course: topStudent.course,
          averageGrade: Number(getAverageGrade(topStudent).toFixed(2)),
        }
      : null,
    courseBreakdown: getCourseAverages(students),
  };
}

// ==========================================
// 2. Main Execution
// ==========================================

function main() {
  const filePath = path.join(__dirname, 'students.json');
  let students = [];

  // Load from disk using fs.readFileSync or create sample if missing
  try {
    if (fs.existsSync(filePath)) {
      const rawData = fs.readFileSync(filePath, 'utf8');
      students = JSON.parse(rawData);
    } else {
      students = [
        { id: 1, name: 'Alice Smith', year: 2, course: 'Computer Science', grades: [88, 92, 95], enrolled: true },
        { id: 2, name: 'Bob Jones', year: 3, course: 'Mathematics', grades: [78, 85, 80], enrolled: true },
        { id: 3, name: 'Charlie Brown', year: 1, course: 'Computer Science', grades: [90, 87, 93], enrolled: false },
        { id: 4, name: 'Diana Prince', year: 4, course: 'Physics', grades: [98, 96, 100], enrolled: true },
        { id: 5, name: 'Evan Wright', year: 2, course: 'Mathematics', grades: [], enrolled: false }
      ];
      fs.writeFileSync(filePath, JSON.stringify(students, null, 2));
    }
  } catch (error) {
    console.error('Error reading students.json:', error.message);
    return;
  }

  console.log('===================================================');
  console.log('              STUDENT RECORDS REPORT               ');
  console.log('===================================================\n');

  // Enrollment Count
  const status = getEnrolledCount(students);
  console.log(`Total Students Loaded: ${students.length}`);
  console.log(`Enrolled: ${status.enrolled} | Not Enrolled: ${status.notEnrolled}\n`);

  // Top Students
  console.log('--- TOP 3 STUDENTS ---');
  const topStudents = getTopStudents(students, 3);
  topStudents.forEach((student, index) => {
    console.log(`${index + 1}. ${student.name} (${student.course}) - Avg: ${getAverageGrade(student).toFixed(2)}`);
  });
  console.log('');

  // Course Averages
  console.log('--- AVERAGE GRADE BY COURSE ---');
  getCourseAverages(students).forEach((item) => {
    console.log(`- ${item.course}: ${item.averageGrade}`);
  });
  console.log('');

  // Student Search
  console.log('--- STUDENT SEARCH TEST ---');
  const found = findStudent(students, 'Alice Smith');
  if (found) {
    console.log(`Found: ${found.name} | ID: ${found.id} | Course: ${found.course}`);
  } else {
    console.log('Student not found.');
  }
  console.log('');

  // Write Summary to Disk
  const summary = exportSummary(students);
  const reportPath = path.join(__dirname, 'report.json');
  fs.writeFileSync(reportPath, JSON.stringify(summary, null, 2));
  console.log(`[SUCCESS] Generated summary report at ${reportPath}\n`);
  console.log('===================================================');
}

main();
