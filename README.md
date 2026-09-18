
const fs = require('fs');
const path = require('path');

// ==========================================
// 1. Core Functions
// ==========================================

/**
 * Calculates the average grade for a single student.
 * Handles students with empty or missing grade arrays safely.
 */
function getAverageGrade(student) {
  if (!student || !Array.isArray(student.grades) || student.grades.length === 0) {
    return 0;
  }
  const sum = student.grades.reduce((acc, curr) => acc + curr, 0);
  return sum / student.grades.length;
}

/**
 * Returns the top n students sorted by average grade in descending order.
 */
function getTopStudents(students, n) {
  if (!Array.isArray(students)) return [];
  if (typeof n !== 'number' || n < 0) {
    throw new Error('Parameter "n" must be a non-negative integer.');
  }

  return [...students] // Avoid mutating the original array
    .sort((a, b) => getAverageGrade(b) - getAverageGrade(a))
    .slice(0, n);
}

/**
 * Groups all students by their enrolled course.
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
 * Returns a count breakdown of enrolled vs. non-enrolled students.
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
 * Performs a case-insensitive search for a student by full name.
 */
function findStudent(students, name) {
  if (!Array.isArray(students) || typeof name !== 'string') return null;

  const targetName = name.trim().toLowerCase();
  return students.find((student) => student.name?.toLowerCase() === targetName) || null;
}

/**
 * Calculates the average grade across all students per course,
 * sorted from highest to lowest.
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
 * Builds a comprehensive summary object for the dataset.
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
// 2. Data Loader & Main Runner
// ==========================================

function loadDataset(filePath) {
  try {
    const fullPath = path.resolve(filePath);
    if (!fs.existsSync(fullPath)) {
      console.warn(`File not found at ${fullPath}. Creating mock dataset...`);
      const sampleData = [
        { id: 1, name: 'Alice Smith', year: 2, course: 'Computer Science', grades: [88, 92, 95], enrolled: true },
        { id: 2, name: 'Bob Jones', year: 3, course: 'Mathematics', grades: [78, 85, 80], enrolled: true },
        { id: 3, name: 'Charlie Brown', year: 1, course: 'Computer Science', grades: [90, 87, 93], enrolled: false },
        { id: 4, name: 'Diana Prince', year: 4, course: 'Physics', grades: [98, 96, 100], enrolled: true },
        { id: 5, name: 'Evan Wright', year: 2, course: 'Mathematics', grades: [], enrolled: false }
      ];
      fs.writeFileSync(fullPath, JSON.stringify(sampleData, null, 2));
      return sampleData;
    }

    const rawData = fs.readFileSync(fullPath, 'utf8');
    return JSON.parse(rawData);
  } catch (error) {
    console.error('Error loading dataset:', error.message);
    return [];
  }
}

function main() {
  const dataPath = path.join(__dirname, 'students.json');
  const students = loadDataset(dataPath);

  console.log('===================================================');
  console.log('              STUDENT RECORDS REPORT               ');
  console.log('===================================================\n');

  // Total and Enrolled Breakdown
  const enrollmentStatus = getEnrolledCount(students);
  console.log(`Total Students Loaded: ${students.length}`);
  console.log(`Enrolled: ${enrollmentStatus.enrolled} | Not Enrolled: ${enrollmentStatus.notEnrolled}\n`);

  // Top Students
  console.log('--- TOP 3 STUDENTS ---');
  const topStudents = getTopStudents(students, 3);
  topStudents.forEach((student, index) => {
    const avg = getAverageGrade(student).toFixed(2);
    console.log(`${index + 1}. ${student.name} (${student.course}) - Avg: ${avg}`);
  });
  console.log('');

  // Course Averages
  console.log('--- AVERAGE GRADE BY COURSE ---');
  const courseAverages = getCourseAverages(students);
  courseAverages.forEach((item) => {
    console.log(`- ${item.course}: ${item.averageGrade}`);
  });
  console.log('');

  // Search Example
  console.log('--- STUDENT SEARCH TEST ---');
  const searchName = 'Alice Smith';
  const foundStudent = findStudent(students, searchName);
  if (foundStudent) {
    console.log(`Found: ${foundStudent.name} | ID: ${foundStudent.id} | Course: ${foundStudent.course}`);
  } else {
    console.log(`Student "${searchName}" not found.`);
  }
  console.log('');

  // Export Summary to JSON File
  const summary = exportSummary(students);
  const outputPath = path.join(__dirname, 'report.json');
  fs.writeFileSync(outputPath, JSON.stringify(summary, null, 2));
  console.log(`[SUCCESS] Summary exported to ${outputPath}\n`);
  console.log('===================================================');
}

// Execute program
main();
