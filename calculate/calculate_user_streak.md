-- Create function to calculate user streak based on answer streak and due streak
create function calculate_user_streak(p_user_id integer, p_event_count integer) returns "UserStreakData"
    language plpgsql
as
$$
DECLARE
    v_streak_count INT := 0;  -- Initialize streak count
    v_last_updated DATE;      -- Variable to store last updated date
    user_streak_data "UserStreakData"%ROWTYPE;  -- Row to store the user's streak data
    v_current_ist_date DATE;  -- Variable for current IST date
    v_prev_ist_date DATE;     -- Variable for the previous IST date
    v_due_streak INT;         -- Variable to store due streak
    v_answer_streak INT;      -- Variable to store answer streak
    v_earliest_date DATE;     -- Variable to store the earliest date
    v_latest_date DATE;       -- Variable to store the latest date
BEGIN
    -- Get current and previous IST dates
    SELECT date(timezone('Asia/Kolkata', now())) INTO v_current_ist_date;
    v_prev_ist_date := v_current_ist_date - 1;

    -- Get existing streak data row
    SELECT * INTO user_streak_data
    FROM "UserStreakData"
    WHERE "userId" = p_user_id;

    -- Initialize streak count from existing data
    v_streak_count := COALESCE(user_streak_data."streakCount", 0);
    v_last_updated := COALESCE(user_streak_data."lastStreakDate", date(timezone('Asia/Kolkata', user_streak_data."lastStreakUpdated")));

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
    IF v_last_updated > v_earliest_date THEN
        v_earliest_date := v_last_updated;
    END IF;

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
    IF v_last_updated > v_earliest_date THEN
        v_earliest_date := v_last_updated;
    END IF;

    IF v_earliest_date >= v_latest_date THEN
        v_answer_streak := 0;
    ELSE
        v_answer_streak := calculate_user_answer_streak(p_user_id, v_latest_date, v_earliest_date);
    END IF;

    -- Add 1 to streak count only if today's practice exists
    IF v_answer_streak = 1 THEN
        v_streak_count := v_streak_count + 1;
    END IF;

    -- Update the streak data row
    user_streak_data."userId" := p_user_id;
    user_streak_data."streakCount" := v_streak_count;
    user_streak_data."updatedAt" := current_timestamp;
    user_streak_data."createdAt" := COALESCE(user_streak_data."createdAt", user_streak_data."updatedAt");
    user_streak_data."lastStreakDate" := CASE
        WHEN v_answer_streak = 1 THEN v_current_ist_date
        ELSE v_prev_ist_date
    END;
    user_streak_data."lastStreakUpdated" := user_streak_data."lastStreakDate";

    -- Return the user's streak data
    RETURN user_streak_data;
END;
$$;

-- Alter function owner
alter function calculate_user_streak(integer, integer) owner to learner;
