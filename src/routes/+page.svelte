<script>
    const numberOfStudents = 16;
    const numberOfGroups = 6;
    const animals = [
        "whale",
        "tiger",
        "elephant",
        "gibbon",
        "ibex",
        "sloth",
        "giraffe",
        "kangaroo",
        "penguin",
        "fennec fox",
    ];
    // Expected group structure for 16 students in 6 groups: 4 groups of 3, 2 groups of 2
    const targetGroupCounts = { trio: 4, duo: 2 };

    const studentNames = [
        "Alice",
        "Ariella",
        "Charlie",
        "Christine",
        "Cici",
        "Eddie",
        "Grace",
        "Howard",
        "Jasper",
        "Jerry D",
        "Jerry P",
        "Lisa WY",
        "Lisa ZL",
        "Milly",
        "Miriel",
        "Queenie",
    ].sort(() => Math.random() - 0.5);

    function getRandomChoices(animalList) {
        const shuffledAnimals = [...animalList].sort(() => Math.random() - 0.5);
        return {
            choice1: shuffledAnimals[0],
            choice2: shuffledAnimals[1],
            choice3: shuffledAnimals[2],
        };
    }

    let students = $state(
        Array(numberOfStudents)
            .fill(null)
            .map((_, i) => {
                const randomChoices = getRandomChoices(animals);
                return {
                    id: i,
                    name: studentNames[i],
                    choice1: randomChoices.choice1,
                    choice2: randomChoices.choice2,
                    choice3: randomChoices.choice3,
                };
            }),
    );

    let finalGroups = $state([]);
    let errorMessage = $state("");
    let isProcessing = $state(false);

    function validateChoices() {
        errorMessage = "";
        for (const student of students) {
            if (!student.name.trim()) {
                errorMessage = `Please enter a name for all students (Student ID ${student.id + 1}).`;
                return false;
            }
            if (!student.choice1 || !student.choice2 || !student.choice3) {
                errorMessage = `Student ${student.name} must make all 3 choices.`;
                return false;
            }
            if (
                student.choice1 === student.choice2 ||
                student.choice1 === student.choice3 ||
                student.choice2 === student.choice3
            ) {
                errorMessage = `Student ${student.name} must select three *different* animals.`;
                return false;
            }
        }
        return true;
    }

    function createGroups() {
        if (!validateChoices()) {
            return;
        }
        isProcessing = true;
        finalGroups = []; // Reset groups
        setTimeout(() => {
            try {
                // --- Grouping Algorithm ---
                let studentsToAssign = JSON.parse(JSON.stringify(students)); // Deep copy
                let formedGroups = [];
                const getStudentsByChoice = (animal, rank) => {
                    return studentsToAssign.filter(
                        (s) => s[`choice${rank}`] === animal,
                    );
                };
                const removeStudents = (studentIds) => {
                    studentsToAssign = studentsToAssign.filter(
                        (s) => !studentIds.includes(s.id),
                    );
                };

                // Track assigned animals to ensure uniqueness
                let assignedAnimals = new Set();

                // Helper: Get an unused animal (either from preferences or randomly)
                const getUnusedAnimal = (preferredAnimals) => {
                    for (const animal of preferredAnimals) {
                        if (!assignedAnimals.has(animal)) {
                            return animal;
                        }
                    }
                    // Fallback: Return a random unused animal
                    const unusedAnimals = animals.filter(
                        (animal) => !assignedAnimals.has(animal),
                    );
                    return unusedAnimals[
                        Math.floor(Math.random() * unusedAnimals.length)
                    ];
                };

                // --- Phase 1: Form groups based on 1st Choices ---
                let groupsMadeCount = { trio: 0, duo: 0 };
                for (const animal of animals) {
                    if (assignedAnimals.has(animal)) continue; // Skip already assigned animals

                    const firstChoosers = getStudentsByChoice(animal, 1);

                    // Try to form trios first
                    while (
                        firstChoosers.length >= 3 &&
                        groupsMadeCount.trio < targetGroupCounts.trio
                    ) {
                        const groupMembers = firstChoosers.splice(0, 3);
                        formedGroups.push({
                            animal: animal,
                            students: groupMembers,
                        });
                        removeStudents(groupMembers.map((s) => s.id));
                        groupsMadeCount.trio++;
                        assignedAnimals.add(animal); // Mark animal as used
                    }

                    // Then try to form duos
                    while (
                        firstChoosers.length >= 2 &&
                        groupsMadeCount.duo < targetGroupCounts.duo
                    ) {
                        const groupMembers = firstChoosers.splice(0, 2);
                        formedGroups.push({
                            animal: animal,
                            students: groupMembers,
                        });
                        removeStudents(groupMembers.map((s) => s.id));
                        groupsMadeCount.duo++;
                        assignedAnimals.add(animal); // Mark animal as used
                    }
                }

                // --- Phase 2: Augment Duos to Trios (using best available choices) ---
                for (let i = 0; i < formedGroups.length; i++) {
                    const group = formedGroups[i];
                    if (
                        group.students.length === 2 &&
                        groupsMadeCount.trio < targetGroupCounts.trio
                    ) {
                        let foundThirdMember = false;

                        // Try finding someone with this animal as 1st choice
                        let potentialAdds = getStudentsByChoice(
                            group.animal,
                            1,
                        );
                        if (potentialAdds.length > 0) {
                            const studentToAdd = potentialAdds[0];
                            group.students.push(studentToAdd);
                            removeStudents([studentToAdd.id]);
                            groupsMadeCount.duo--;
                            groupsMadeCount.trio++;
                            foundThirdMember = true;
                        }

                        // If not found, try 2nd choice
                        if (!foundThirdMember) {
                            potentialAdds = getStudentsByChoice(
                                group.animal,
                                2,
                            );
                            if (potentialAdds.length > 0) {
                                const studentToAdd = potentialAdds[0];
                                group.students.push(studentToAdd);
                                removeStudents([studentToAdd.id]);
                                groupsMadeCount.duo--;
                                groupsMadeCount.trio++;
                                foundThirdMember = true;
                            }
                        }

                        // If not found, try 3rd choice
                        if (!foundThirdMember) {
                            potentialAdds = getStudentsByChoice(
                                group.animal,
                                3,
                            );
                            if (potentialAdds.length > 0) {
                                const studentToAdd = potentialAdds[0];
                                group.students.push(studentToAdd);
                                removeStudents([studentToAdd.id]);
                                groupsMadeCount.duo--;
                                groupsMadeCount.trio++;
                            }
                        }
                    }
                }

                // --- Phase 3: Form remaining groups with leftovers ---
                const createRemainingGroup = (size) => {
                    if (studentsToAssign.length < size) return false; // Not enough students left

                    // Extract students for the new group
                    const groupMembers = studentsToAssign.slice(0, size);

                    // Find the best matching animal for the group
                    const animalFrequency = {};
                    for (let rank = 1; rank <= 3; rank++) {
                        for (const animal of animals) {
                            const choosers = getStudentsByChoice(animal, rank);
                            const relevantChoosers = choosers.filter(
                                (student) =>
                                    groupMembers.some(
                                        (member) => member.id === student.id,
                                    ),
                            );
                            relevantChoosers.forEach(() => {
                                animalFrequency[animal] =
                                    (animalFrequency[animal] || 0) + 1;
                            });
                        }
                    }

                    // Determine the most frequent animal (unused)
                    let bestMatchAnimal = null;
                    let maxCount = 0;
                    for (const [animal, count] of Object.entries(
                        animalFrequency,
                    )) {
                        if (count > maxCount && !assignedAnimals.has(animal)) {
                            bestMatchAnimal = animal;
                            maxCount = count;
                        }
                    }

                    // Fallback: Use an unused animal
                    if (!bestMatchAnimal) {
                        bestMatchAnimal = getUnusedAnimal(
                            groupMembers.flatMap((student) => [
                                student.choice1,
                                student.choice2,
                                student.choice3,
                            ]),
                        );
                    }

                    // Assign the group with the best match animal
                    formedGroups.push({
                        animal: `Mixed (Best Match: ${bestMatchAnimal})`,
                        students: groupMembers,
                    });

                    // Remove assigned students from the pool
                    removeStudents(groupMembers.map((s) => s.id));
                    assignedAnimals.add(bestMatchAnimal); // Mark animal as used
                    return true;
                };

                // Create remaining trios needed
                while (
                    groupsMadeCount.trio < targetGroupCounts.trio &&
                    studentsToAssign.length >= 3
                ) {
                    if (createRemainingGroup(3)) {
                        groupsMadeCount.trio++;
                    } else {
                        break;
                    }
                }

                // Create remaining duos needed
                while (
                    groupsMadeCount.duo < targetGroupCounts.duo &&
                    studentsToAssign.length >= 2
                ) {
                    if (createRemainingGroup(2)) {
                        groupsMadeCount.duo++;
                    } else {
                        break;
                    }
                }

                // --- Phase 4: Final Assignment/Correction (Handle edge cases if counts are off) ---
                let studentIndex = 0;
                while (studentIndex < studentsToAssign.length) {
                    let assigned = false;
                    const student = studentsToAssign[studentIndex];
                    for (const group of formedGroups) {
                        if (group.students.length === 2) {
                            group.students.push(student);
                            studentsToAssign.splice(studentIndex, 1);
                            assigned = true;
                            break;
                        }
                    }
                    if (!assigned) {
                        console.error("Could not assign student:", student);
                        errorMessage =
                            "Error during final assignment. Please check inputs or algorithm logic.";
                        studentIndex++;
                    }
                }

                // Verify final count
                if (
                    formedGroups.length !== numberOfGroups ||
                    studentsToAssign.length > 0
                ) {
                    console.error(
                        "Final group count or unassigned students mismatch.",
                        formedGroups,
                        studentsToAssign,
                    );
                    errorMessage = `Grouping failed to produce exactly ${numberOfGroups} groups with all students assigned. (${formedGroups.length} groups formed, ${studentsToAssign.length} students left). Check algorithm logic.`;
                    finalGroups = [];
                } else {
                    // Check final group sizes
                    let finalTrioCount = 0;
                    let finalDuoCount = 0;
                    for (const group of formedGroups) {
                        if (group.students.length === 3) finalTrioCount++;
                        else if (group.students.length === 2) finalDuoCount++;
                        else {
                            errorMessage = `Error: Group with invalid size ${group.students.length} created.`;
                            finalGroups = [];
                            isProcessing = false;
                            return;
                        }
                    }
                    if (
                        finalTrioCount !== targetGroupCounts.trio ||
                        finalDuoCount !== targetGroupCounts.duo
                    ) {
                        errorMessage = `Error: Final group structure is incorrect (${finalTrioCount} trios, ${finalDuoCount} duos). Expected ${targetGroupCounts.trio} trios, ${targetGroupCounts.duo} duos.`;
                        finalGroups = [];
                    } else {
                        finalGroups = formedGroups; // Success!
                    }
                }
            } catch (e) {
                console.error("Error during grouping:", e);
                errorMessage = `An unexpected error occurred: ${e.message}`;
                finalGroups = [];
            } finally {
                isProcessing = false;
            }
        }, 50); // Short delay
    }
</script>

<svelte:head>
    <title>Student Grouping App</title>
</svelte:head>

<main class="container">
    <h1>Student Animal Preference Grouping</h1>
    <p>
        Enter student names and their 1st, 2nd, and 3rd choice for an animal.
        The system will create {numberOfGroups} groups (4 groups of 3, 2 groups of
        2) based on preferences.
    </p>

    <section class="input-section">
        <h2>Student Choices</h2>
        <div class="student-inputs">
            {#each students as student, i (student.id)}
                <div class="student-card">
                    <strong>Student {i + 1}:</strong>
                    <input
                        type="text"
                        placeholder="Enter Name"
                        bind:value={students[i].name}
                    />
                    <select bind:value={students[i].choice1}>
                        <option value="" disabled>1st Choice</option>
                        {#each animals as animal}
                            <option value={animal}>{animal}</option>
                        {/each}
                    </select>
                    <select bind:value={students[i].choice2}>
                        <option value="" disabled>2nd Choice</option>
                        {#each animals as animal}
                            <option
                                value={animal}
                                disabled={students[i].choice1 === animal}
                                >{animal}</option
                            >
                        {/each}
                    </select>
                    <select bind:value={students[i].choice3}>
                        <option value="" disabled>3rd Choice</option>
                        {#each animals as animal}
                            <option
                                value={animal}
                                disabled={students[i].choice1 === animal ||
                                    students[i].choice2 === animal}
                                >{animal}</option
                            >
                        {/each}
                    </select>
                </div>
            {/each}
        </div>
    </section>

    <section class="controls">
        <button onclick={createGroups} disabled={isProcessing}>
            {#if isProcessing}
                Processing...
            {:else}
                Create Groups
            {/if}
        </button>
        {#if errorMessage}
            <p class="error">{errorMessage}</p>
        {/if}
    </section>

    {#if finalGroups.length > 0}
        <section class="results-section">
            <h2>Generated Groups</h2>
            <div class="group-outputs">
                {#each finalGroups as group, i}
                    <div class="group-card">
                        <h3>
                            Group {i + 1}
                            {#if group.animal}<span
                                    >(Focus: {group.animal})</span
                                >{/if}
                        </h3>
                        <ul>
                            {#each group.students as student}
                                <li>{student.name}</li>
                            {/each}
                        </ul>
                    </div>
                {/each}
            </div>
        </section>
    {/if}
</main>

<style>
    /* Styles remain the same as previously provided */
    .container {
        font-family: sans-serif;
        max-width: 1200px;
        margin: 2em auto;
        padding: 1em;
    }

    .input-section h2,
    .results-section h2 {
        border-bottom: 2px solid #eee;
        padding-bottom: 0.5em;
        margin-bottom: 1em;
    }

    .student-inputs,
    .group-outputs {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
        gap: 1em;
    }

    .student-card,
    .group-card {
        border: 1px solid #ccc;
        border-radius: 8px;
        padding: 1em;
        background-color: #f9f9f9;
    }

    .student-card strong {
        display: block;
        margin-bottom: 0.5em;
    }

    .student-card input,
    .student-card select {
        display: block;
        width: 95%; /* Slightly less than 100% to prevent overflow */
        margin-bottom: 0.5em;
        padding: 8px;
        border: 1px solid #ccc;
        border-radius: 4px;
    }

    .controls {
        margin: 2em 0;
        text-align: center;
    }

    .controls button {
        padding: 12px 25px;
        font-size: 1.1em;
        background-color: #007bff;
        color: white;
        border: none;
        border-radius: 5px;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }

    .controls button:hover {
        background-color: #0056b3;
    }

    .controls button:disabled {
        background-color: #aaa;
        cursor: not-allowed;
    }

    .error {
        color: #d9534f; /* Bootstrap danger color */
        margin-top: 1em;
        font-weight: bold;
    }

    .group-card h3 {
        margin-top: 0;
        color: #333;
        font-size: 1.1em;
    }
    .group-card h3 span {
        /* Corrected selector */
        font-size: 0.9em;
        font-weight: normal;
        color: #555;
        margin-left: 5px; /* Added margin for spacing */
    }

    .group-card ul {
        list-style: none;
        padding-left: 0;
    }
    .group-card li {
        background-color: #e9ecef;
        padding: 5px 8px;
        margin-bottom: 5px;
        border-radius: 3px;
    }
</style>
