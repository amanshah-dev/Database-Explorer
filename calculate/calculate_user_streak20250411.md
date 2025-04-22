-- Create function to calculate the user's streak for a given user and event count (specific to 2025-04-11)
create function calculate_user_streak_20250411(p_user_id integer, p_event_count integer) returns integer
    language plpgsql
as
$$
DECLARE
    v_streak_count INT := 0;               -- Initialize streak count
    v_last_updated DATE;                    -- Variable to store last updated date
    v_current_ist_date DATE;                -- Variable for current IST date
    v_prev_ist_date DATE;                   -- Variable for the previous IST date
    v_due_streak INT;                       -- Variable to store due streak
    v_answer_streak INT;                    -- Variable to store answer streak
    v_earliest_date DATE;                   -- Variable to store the earliest date
    v_latest_date DATE;                     -- Variable to store the latest date
BEGIN
    -- Get current and previous IST dates
    SELECT date(timezone('Asia/Kolkata', now())) INTO v_current_ist_date;
    v_prev_ist_date := v_current_ist_date - 1;

    -- Get existing streak data
    SELECT COALESCE("streakCount", 0), COALESCE("lastStreakDate", date(timezone('Asia/Kolkata', "lastStreakUpdated")))
    INTO v_streak_count, v_last_updated
    FROM "UserStreakData"
    WHERE "userId" = p_user_id;

    -- Calculate DUE-based streak for period ending 3 days ago
    v_latest_date := v_prev_ist_date - 3;
    v_earliest_date := COALESCE(v_last_updated, v_latest_date - INTERVAL '2 years');

    IF v_earliest_date >= v_latest_date THEN
        v_due_streak := 0;
    ELSE
        v_due_streak := calculate_user_due_streak(p_user_id, v_latest_date, v_earliest_date);
    END IF;

    -- Conditionally add existing streak count
    IF (v_earliest_date < v_latest_date) AND (v_due_streak = (v_latest_date - v_earliest_date)) THEN
        v_streak_count := COALESCE(v_streak_count, 0) + v_due_streak;
    ELSIF v_earliest_date < v_latest_date THEN
        v_streak_count := v_due_streak;
    END IF;

    -- Calculate answer-based streak for recent 3 days
    v_latest_date := v_prev_ist_date;
    v_earliest_date := v_prev_ist_date - 3;

    IF v_earliest_date >= v_latest_date THEN
        v_answer_streak := 0;
    ELSE
        v_answer_streak := calculate_user_answer_streak(p_user_id, v_latest_date, v_earliest_date);
    END IF;

    -- Conditionally add accumulated streak
    IF (v_earliest_date < v_latest_date) AND (v_answer_streak = (v_latest_date - v_earliest_date)) THEN
        v_streak_count := v_streak_count + v_answer_streak;
    ELSIF v_earliest_date < v_latest_date THEN
        v_streak_count := v_answer_streak;
    END IF;
 
    -- Calculate answer-based streak for previous day
    v_latest_date := v_current_ist_date;
    v_earliest_date := v_prev_ist_date;
 
    v_answer_streak := calculate_user_answer_streak(p_user_id, v_latest_date, v_earliest_date);
 
    -- Add 1 to streak count only if today's practice exists
    IF v_answer_streak = 1 THEN
        v_streak_count := v_streak_count + 1;
    END IF;
 
    -- Return the final streak count
    RETURN v_streak_count;
END;
$$;

-- Alter function owner
alter function calculate_user_streak_20250411(integer, integer) owner to learner;
