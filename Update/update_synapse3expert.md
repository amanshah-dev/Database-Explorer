create procedure update_synapse3expert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Synapse III', achievementCategoryId, userIds);
END;
$$;

alter procedure update_synapse3expert(bigint, bigint[]) owner to learner;
