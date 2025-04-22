-- Create function to calculate section durations based on the test name and lock status
create function calculate_sections_duration("durationInMin" integer, sections json, "lockSections" boolean, test_name character varying) returns jsonb
    immutable
    language plpgsql
as
$$
DECLARE
  result JSONB := '[]'::JSONB;  -- Initialize the result as an empty JSON array
  section_index INT;             -- Variable to iterate through the sections
  section_duration INT;          -- Duration of each section
  last_section_duration INT;     -- Duration for the last section
  section_count INT;             -- Total number of sections
BEGIN
  -- Check if sections are locked and sections is not NULL
  IF "lockSections" AND sections IS NOT NULL THEN

    -- Determine the number of sections from the input JSON
    section_count := json_array_length(sections);

    -- Set the section duration based on the test name
    IF test_name ILIKE '%NEET%' THEN
      section_duration := 42;  -- Use 42 minutes for NEET PG
    ELSIF test_name ILIKE '%INICET%' OR test_name ILIKE '%AIIMS%' THEN
      section_duration := 45;  -- Use 45 minutes for INICET or AIIMS
    ELSE
      section_duration := 42;  -- Default to 42 minutes
    END IF;

    -- Calculate section durations based on the test type and section count
    IF test_name ILIKE '%INICET%' OR test_name ILIKE '%AIIMS%' THEN
      -- For INICET or AIIMS, set each section to 45 mins and adjust the last section
      FOR section_index IN 0 .. section_count - 2 LOOP
        result := result || jsonb_build_array(jsonb_build_array(
                    section_index * section_duration,
                    (section_index + 1) * section_duration
                  ));
      END LOOP;
      -- Set the last section with the remaining duration
      last_section_duration := "durationInMin" - ((section_count - 1) * section_duration);
      result := result || jsonb_build_array(jsonb_build_array(
                  (section_count - 1) * section_duration,
                  (section_count - 1) * section_duration + last_section_duration
                ));
    ELSE
      -- For NEET PG or other tests with sections of 42 minutes each
      FOR section_index IN 0 .. section_count - 2 LOOP
        result := result || jsonb_build_array(jsonb_build_array(
                    section_index * section_duration,
                    (section_index + 1) * section_duration
                  ));
      END LOOP;

      -- Set the last section duration correctly
      last_section_duration := "durationInMin" - ((section_count - 1) * section_duration);
      result := result || jsonb_build_array(jsonb_build_array(
                  (section_count - 1) * section_duration,
                  (section_count - 1) * section_duration + last_section_duration
                ));
    END IF;
  END IF;

  -- Return the result containing section durations
  RETURN result;
END;
$$;

-- Alter function owner
alter function calculate_sections_duration(integer, json, boolean, varchar) owner to neetprep_rw;
