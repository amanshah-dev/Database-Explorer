-- Create function to calculate section durations based on given duration and sections
create function calculate_sections_duration("durationInMin" integer, sections json, "lockSections" boolean) returns jsonb
    immutable
    language plpgsql
as
$$
DECLARE
  result JSONB := '[]'::JSONB;   -- Initialize the result as an empty JSON array
  section_index INT;              -- Variable to iterate through sections
  section_duration INT := 42;     -- Fixed duration for each section
  total_sections INT;             -- Total number of sections
  last_section_duration INT;     -- Duration for the last section
BEGIN
  IF "lockSections" AND "sections" IS NOT NULL THEN
    total_sections := "durationInMin" / section_duration;           -- Calculate the total sections
    last_section_duration := "durationInMin" % section_duration;   -- Calculate the remaining duration for the last section

    -- Loop through sections and create duration ranges
    FOR section_index IN 0 .. total_sections - 1 LOOP
      result := result || jsonb_build_array(jsonb_build_array(
                  section_index * section_duration, 
                  (section_index + 1) * section_duration
                ));
    END LOOP;

    -- Handle the last section with the remaining duration
    IF last_section_duration > 0 THEN
      result := result || jsonb_build_array(jsonb_build_array(
                  total_sections * section_duration,
                  total_sections * section_duration + last_section_duration
                ));
    END IF;
  END IF;

  -- Return the result containing the section durations
  RETURN result;
END;
$$;

-- Alter function owner
alter function calculate_sections_duration(integer, json, boolean) owner to neetprep_rw;
