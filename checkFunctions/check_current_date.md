-- Create function to check the current date based on input parameters
create function check_current_date(input_params jsonb) returns boolean
    language plpgsql
as
$$
DECLARE
    current_date_in_india DATE;
    day_part INTEGER;
    remainder INTEGER;
    param_date INTEGER;
    param_divisor INTEGER;
BEGIN
    -- Extract the 'date' and 'divisor' parameters from input_params JSONB
    param_date := (input_params->>'date')::INTEGER;
    param_divisor := (input_params->>'divisor')::INTEGER;

    -- Get the current date in India (Asia/Kolkata timezone)
    current_date_in_india := (CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::DATE;

    -- Extract the day part (day of the month)
    day_part := EXTRACT(DAY FROM current_date_in_india);

    -- Calculate the remainder when the day part is divided by the divisor
    remainder := day_part % param_divisor;

    -- Return true if the remainder equals the provided 'date' parameter
    RETURN remainder = param_date;
END;
$$;

-- Alter function owner
alter function check_current_date(jsonb) owner to neetprep_rw;
